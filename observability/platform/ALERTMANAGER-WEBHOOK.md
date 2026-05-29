# Alertmanager → MC Plan Webhook (PB.13)

MC Backend exposes `POST /api/v1/alerts/alertmanager` which turns each firing
Alertmanager alert into a PB v2 **Plan** (`kind=alert`) with Acknowledge /
Snooze 1h / Open in MC actions. Resolved alerts auto-close the matching open
plan and strip Discord buttons.

Source: `apps/mission-control/mission-control-backend/src/api/routes/alertmanager.ts`

## Endpoint

| Item | Value |
|------|-------|
| URL (in-cluster) | `http://mission-control-backend.mission-control.svc.cluster.local:3000/api/v1/alerts/alertmanager` |
| Method | `POST` |
| Auth | Optional `Authorization: Bearer ${ALERTMANAGER_WEBHOOK_TOKEN}` (constant-time compare; warning logged at boot if unset) |
| Idempotency | Per-alert `fingerprint` is matched against active alertmanager-source plans; re-fires while a plan is open are skipped |
| Default expiry | 24h (plan auto-expires via PB.8 sweep) |

## Alertmanager receiver config

```yaml
receivers:
  - name: mc-plan-service
    webhook_configs:
      - url: http://mission-control-backend.mission-control.svc.cluster.local:3000/api/v1/alerts/alertmanager
        send_resolved: true
        max_alerts: 0   # send all alerts in the group
        http_config:
          # Only needed if ALERTMANAGER_WEBHOOK_TOKEN is set on MC backend.
          authorization:
            type: Bearer
            credentials_file: /etc/alertmanager/secrets/mc-webhook-token/token
```

## Example route config

```yaml
route:
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: mc-plan-service        # default sink for everything
  routes:
    - matchers:
        - severity = "critical"
      receiver: mc-plan-service
      group_wait: 10s              # faster fan-out for critical
      repeat_interval: 1h
    - matchers:
        - severity =~ "warning|info"
      receiver: mc-plan-service
```

Severity mapping inside MC:
- `critical` → Plan `critical`
- `warning` / `warn` → Plan `warn`
- `info` / `informational` → Plan `info`
- missing → Plan `info`
- anything else → Plan `error`

## Alerts that should hit this receiver

Codified targets (per RETRO.2, absorbed into PB.13):

| Alert | Target | Notes |
|-------|--------|-------|
| `NexusBlackboxDown` | Nexus / `npm install` egress for `@petedio/*` | Already partially probed |
| `QbitVpnNatPmpDead` | LXC 110 `qbittorrent-vpn` | The 2026-05-18 silent stall that started this whole story |
| `CloudflareTunnelUnreachable` | `*.pdlab.dev` egress | Pair with [feedback_cloudflare_tunnel_router_first.md](../../../../knowledge/feedback_cloudflare_tunnel_router_first.md) playbook |
| `ArgoCDAppDegraded` / `ArgoCDAppOutOfSync` | Any tracked Application | Use `app` label as `service` |
| `LXCAgentHealthzFailing` | One of ops-investigator / knowledge-janitor / workstation-agent / infra-agent / research-agent / code-review-agent / memory-agent | Pair with `instance` label |

Label conventions assumed by the webhook:
- `severity` (`critical` / `warning` / `info`) — drives Plan severity
- `service` — preferred Plan `target`
- `instance` — fallback `target` if `service` is missing
- `alertname` — final fallback
- annotation `summary` (preferred) or `description` → Plan `summary`
- annotation `runbook_url` is preserved in `sourceMetadata.annotations`

## Cluster wiring status

Shipped in this directory (PB.13/PB.16, 2026-05-28):

- [x] **AlertmanagerConfig CR** — `mc-plan-alertmanager-config.yaml`. Routes
      alerts labelled `notify: mc-plan` to the `mc-plan-service` webhook
      receiver. Auth uses `httpConfig.authorization.credentials`
      (secretKeyRef) rather than the `credentials_file` mount above — the
      operator resolves the secret from the CR's namespace, so no
      `alertmanagerSpec.secrets[]` mount (and thus no out-of-band Helm-values
      edit) is required. The operator's default `OnNamespace` matcher strategy
      prepends `namespace=observability`, so opted-in rules live in this ns.
- [x] **Sealed secret** — `mc-webhook-am-token-sealedsecret.yaml` (key `token`,
      observability ns), holding the same value as
      `mission-control/mc-webhook-secrets:ALERTMANAGER_WEBHOOK_TOKEN` (sealed
      per SEC.9). The MC-backend half is consumed via `envFrom`; this is the
      Alertmanager half.
- [x] **PrometheusRule (real, in-cluster)** — `mc-plan-alerts.yaml`:
      `MCBackendDown`, `MCNotificationServiceDown`, `MCArgoCDUnreachable`,
      `WebSearchServiceDown`. These fire on series that exist today and route
      end-to-end to a Discord card.

Still out of scope / follow-ups:

- [ ] **The 5 aspirational alerts** (`NexusBlackboxDown`, `QbitVpnNatPmpDead`,
      `CloudflareTunnelUnreachable`, `ArgoCDAppDegraded`/`OutOfSync`,
      `LXCAgentHealthzFailing`) have **no metric source today** — recon
      2026-05-28 found no blackbox-exporter, ArgoCD is unscraped
      (`argocd_app_info` empty), nothing scrapes the LXC agents on `.113`, and
      there's no qbit/gluetun exporter. Each needs its scrape/probe target
      stood up first (tracked as OBS.* follow-on tasks). Once their series
      exist, add the rules here with the `notify: mc-plan` label and they route
      automatically.
- [ ] **Capture kube-prometheus-stack Helm values into gitops** + set
      `alertmanagerConfigMatcherStrategy: None` so alerts that legitimately
      carry a non-`observability` namespace can also route via this CR. The
      Helm release is currently installed out-of-band (not reproducible from
      this repo).
- [ ] Snooze proper: the current `snooze_1h` action dismisses the plan instead
      of re-presenting after 1h. Track as a retro candidate; requires a new
      lifecycle hook (e.g., a delayed `pending` re-entry or a snooze table).

## Verification path (when cluster wiring lands)

1. Port-forward MC backend: `kubectl -n mission-control port-forward svc/mission-control-backend 3000:3000`.
2. Send a synthetic firing alert via `curl`:
   ```bash
   curl -sS -X POST http://localhost:3000/api/v1/alerts/alertmanager \
     -H 'content-type: application/json' \
     -H "authorization: Bearer ${ALERTMANAGER_WEBHOOK_TOKEN}" \
     -d '{
       "version":"4","status":"firing","receiver":"mc-plan-service","alerts":[{
         "status":"firing",
         "labels":{"alertname":"SyntheticTest","severity":"warning","service":"mc-backend"},
         "annotations":{"summary":"PB.13 wiring test"},
         "startsAt":"2026-05-23T00:00:00Z","fingerprint":"deadbeefcafe"
       }]
     }'
   ```
   Expect `{ ok: true, processed: 1, created: 1, ... }` and a Plan visible at
   `GET /api/v1/plans?source=alertmanager`.
3. Re-send the same payload — should return `{ ok: true, created: 0, skipped: 1, ... }`.
4. Send the same payload with `status:"resolved"` and `endsAt` — Plan flips to
   `resolved` and Pete Bot is asked to strip buttons.
