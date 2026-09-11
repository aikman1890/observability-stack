# Observability Stack

Production-grade observability configuration from a 20+ year SRE career
(Verizon, JPMorgan Chase, IBM, Wendy's, NTT Data). This repo holds the
alerting rules, dashboards, and analysis queries I actually use — with
symptom-based alerting throughout: pages fire on user-facing symptoms
(SLO burn, latency, error rate), not on causes (CPU at 90%).

Results this stack has delivered:
- **35% lower MTTD** (mean time to detect) via multiwindow burn-rate alerts and error-rate spike detection
- **40% MTTR cut** via MTTR-per-service tracking and noise reduction
- **85% downtime reduction** via faster detection + tighter runbooks
- **5-minute outage response** via CloudWatch alarms on critical paths

## Layout

```
prometheus/rules/      Alerting rules (load into Prometheus or Thanos/Ruler)
  error-budget-burn.yml   Multiwindow multi-burn-rate alerts (Google SRE workbook pattern)
  node-saturation.yml     Node CPU/memory/disk saturation (ticket-level, not page-level)
  latency.yml             p99 histogram latency alerts on RED metrics
grafana/dashboards/
  red-method.json         Complete RED dashboard (Rate/Errors/Duration), importable
splunk/
  mttr-queries.spl        MTTR per service, P1 triage trend, alert-fatigue top 10,
                          error-rate spike detection (the query behind the 35% MTTD cut)
dynatrace/
  slo-configuration.md    Burn-rate SLOs, Davis AI wiring, PagerDuty/Opsgenie
                          notification, management-zone scoping — from the field
```

## Loading the rules

Prometheus (alertmanager + rule files):

```yaml
# prometheus.yml
rule_files:
  - "rules/error-budget-burn.yml"
  - "rules/node-saturation.yml"
  - "rules/latency.yml"
```

Validate before deploying:

```bash
promtool check rules prometheus/rules/*.yml
promtool test rules tests/*.yml   # if you add unit tests for your rules
```

For Thanos Ruler or Cortex, the files load unchanged.

## Importing the dashboard

1. Grafana → Dashboards → Import → Upload `grafana/dashboards/red-method.json`
2. Select your Prometheus datasource when prompted.
3. The dashboard uses template variables `$namespace` and `$service`; it assumes
   RED metrics exposed as `http_requests_total` and
   `http_request_duration_seconds_bucket` with `namespace`, `service`, and
   `code` labels. Rename label matchers if your instrumentation differs.

## Alerting philosophy: symptoms, not causes

| Pages (wake someone up) | Tickets (fix during the day) |
|---|---|
| Error-budget fast burn (1h/6h windows) | Node CPU/memory/disk saturation |
| p99 latency breach vs SLO | Slow burn (6h/3d windows) |
| Error-rate spike (MTTD detector) | Top-10 noisiest alerts review |

Why: CPU at 90% is not a user-facing symptom. A page should mean "users are
hurting or about to hurt." Everything else is a ticket with a runbook link.
This is how we cut alert fatigue and got to a 5-minute response on real outages.

## Conventions

- All rules carry `runbook_url` annotations pointing at the sibling
  [sre-runbooks](../sre-runbooks) repo — an alert without a runbook is just noise.
- Recording rules live alongside the alerts they feed; keep evaluation
  intervals at 1m and `for:` durations generous enough to avoid flapping.
- Thresholds here are starting points tuned for a 99.9% availability SLO.
  Adjust burn-rate windows to your own SLOs (see `../sre-runbooks/sli-slo/`).

## License

MIT — see [LICENSE](LICENSE).
