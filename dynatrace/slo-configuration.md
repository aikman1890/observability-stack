# Dynatrace SLO Configuration — Field Guide

How I configure burn-rate SLOs, Davis AI anomaly detection, problem
notification, and management-zone scoping in Dynatrace. Written from
production experience running Dynatrace alongside Prometheus/Splunk in
large environments (finance, telecom).

## 1. Burn-rate SLOs (not just "availability %")

Dynatrace's SLO model is error-budget based, which maps cleanly onto the
same multiwindow burn-rate thinking as the Prometheus rules in this repo.

**Setup path:** Observability → SLOs → Add SLO → Service-level objective.

**What I configure:**

| Setting | Value | Why |
|---|---|---|
| Indicator | Failure rate (or response-time p95/p99) | Failure rate for availability SLOs; latency SLOs get their own SLO so burn reasons stay separable |
| Objective | 99.9% | Matches the Prometheus burn-rate alerts; one SLO definition everywhere |
| Evaluation timeframe | 30 days, rolling | Aligns with the 43.2-minute monthly budget math |
| Burn-rate alerts | Fast burn: 14.4x over 1h; Slow burn: 6x over 6h | Same thresholds as `prometheus/rules/error-budget-burn.yml` — one mental model, two tools |
| Evaluation | Davis AI-assisted baseline | Lets Davis learn weekly seasonality instead of a flat threshold |

**Key practice:** one SLO per service per indicator. A combined "availability
+ latency" SLO hides which leg is burning. Separate SLOs, separate burn
alerts, separate runbooks.

**Error-budget policy I enforce:** below 50% remaining → ticket and review
in the weekly ops meeting; below 25% → freeze non-essential deploys for that
service until the budget recovers. Dynatrace shows this on the SLO tile —
I pin the SLO overview to the team dashboard so it's visible without digging.

## 2. Davis AI anomaly detection wiring

Davis is only as good as the baseline it learns from. Configuration that
actually works:

1. **Let it learn before trusting it.** Davis needs ~2 weeks of steady
   traffic to build a reliable baseline. For the first two weeks after
   onboarding a service, run Davis in *observe* mode: alerts go to a
   low-priority channel, and every false positive gets feedback (mark as
   expected). After two weeks, promote to paging for P1 paths.

2. **Feed it deploy markers.** Connect your CI/CD (Jenkins, GitHub Actions,
   Azure DevOps) so Davis correlates anomalies with releases. In my
   experience, ~60% of latency anomalies trace to a deploy within the hour —
   without the marker, Davis still catches the anomaly but the responder
   wastes 20 minutes finding the cause.

3. **Tune detection sensitivity per service tier:**
   - Tier 0 (payments, auth): High sensitivity — accept some noise.
   - Tier 1 (core APIs): Medium.
   - Tier 2 (internal tools, batch): Low — ticket only.
   
   One global sensitivity setting is how you get either alert fatigue or
   blind spots. There is no middle ground that works for both.

4. **Disable Davis on synthetic-only services.** If a service has no real
   user traffic, Davis learns the synthetic cadence as "normal" and pages
   when the synthetic breaks — which is a monitoring problem, not a user
   problem. Use explicit synthetic monitors with their own thresholds there.

## 3. Problem notification → PagerDuty / Opsgenie

**Setup path:** Settings → Integration → Problem notifications → Add.

**PagerDuty:**
- Integration type: PagerDuty. Paste the integration key from a PagerDuty
  service configured with the Dynatrace integration.
- Alerting profile (see §4) selects *which* problems route to *which*
  PagerDuty service — Tier 0 problems go to the Tier 0 PagerDuty service
  with the aggressive escalation policy; Tier 2 goes to the standard one.
- Include problem details payload: entity tags, impacted service, root
  cause entity, and the Davis AI root-cause summary. The responder should
  be able to triage from the PagerDuty push notification alone.

**Opsgenie:**
- Same pattern via the Opsgenie webhook integration; map Dynatrace problem
  severity → Opsgenie priority (P1→P1, P2→P2, everything else→P3).
- Set `responders` to the owning team's schedule so the right rotation gets
  it — problems routed to a generic queue die in the generic queue.

**What I always include in the notification payload:**
- Service name + environment + management zone
- Davis root-cause entity (e.g. "database slowdown on orders-db")
- Link back to the Dynatrace problem (deep link) and to the runbook
- SLO/burn context if the problem breached an SLO ("error budget at 38%")

**Anti-pattern I keep killing:** routing every Davis anomaly to paging.
Anomaly ≠ problem. Davis anomalies feed the problem-detection engine;
*problems* page. If your team gets paged for raw anomalies, the alerting
profile is misconfigured — fix the profile, don't train people to ignore
the pager.

## 4. Management-zone scoping

Management zones are how you stop teams from drowning in each other's
problems. Rules I follow:

1. **Zone per service team, keyed on entity tags** — e.g. `team:payments`
   applied via OneAgent host tags, K8s namespace labels, or AWS tags.
   Tag at the infrastructure layer (namespace/tag), not by hand in the UI.
2. **Alerting profiles scoped to zones** — each team's alerting profile
   includes only their management zone. A payments deploy never pages the
   search team.
3. **A "platform" zone** for shared infrastructure (K8s clusters, load
   balancers, databases) with its own on-call — so a node problem pages
   platform, not every service team whose pods happened to be on it.
4. **Naming convention:** `<org>-<team>-<env>` (e.g. `acme-payments-prod`).
   Zones get created constantly; without a convention they become unusable
   within a year.

**Ownership rule:** every management zone has exactly one owning team and
one alerting profile. If a zone has no owner, delete it — unowned zones
are where alerts go to be ignored.

## 5. Putting it together — the weekly operating rhythm

- **Monday:** review SLO burn for the week (Dynatrace SLO overview +
  Prometheus slow-burn alerts). Any service under 50% budget gets a ticket.
- **Monthly:** top-10 noisiest alerts review (see `splunk/mttr-queries.spl`
  query 3). Tune thresholds or fix the underlying flakiness.
- **Per incident:** Davis root-cause + deploy markers first, runbook second,
  postmortem within 48h (template in the sre-runbooks repo).
- **Quarterly:** re-baseline Davis sensitivity per tier; services change
  traffic shape and the old tuning goes stale.

This is the operating loop that produced the 35% MTTD improvement and the
40% MTTR cut: detection tuned per tier, noise reviewed monthly, and every
page carrying a runbook link.
