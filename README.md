# GitOps safe API rollout

This repo deploys the `api` service through Argo CD and Argo Rollouts. A new version is promoted only when Prometheus reports at least 99% successful HTTP requests. If the canary returns too many 5xx responses, the Rollout analysis fails and Argo Rollouts aborts back to the previous ReplicaSet.

## Pass checklist

- All Kubernetes changes are declared in Git under `k8s-api/` and synced by Argo CD.
- Argo CD can reproduce the desired state from Git with `Synced` and no drift.
- Rollback is a normal Git rollback: `git revert HEAD && git push`.
- Prometheus scrapes `/metrics`, records one SLO metric, and fires one alert when the SLO is violated.
- Argo Rollouts replaces manual pause gates with `AnalysisTemplate`; good canaries promote to 100%, bad canaries are aborted automatically.

## What is included

- GitOps entrypoint: `argocd/root.yaml`
- API application: `argocd/apps/api.yaml`
- Rollout and service: `k8s-api/api.yaml`
- Canary gate: `k8s-api/analysis-template.yaml`
- Metrics scrape: `k8s-api/servicemonitor.yaml`
- SLO alert: `k8s-api/prometheus-rule.yaml`
- Kustomize resource list: `k8s-api/kustomization.yaml`
- Monitoring stack: `argocd/apps/kube-prometheus-stack.yaml`

## Demo commands

Check Argo CD sync:

```powershell
kubectl -n argocd get applications
kubectl -n demo get rollout api
kubectl argo rollouts -n demo get rollout api
```

Confirm the API application is reproduced from Git with no drift:

```powershell
argocd app get api
argocd app diff api
```

Generate traffic so Prometheus has data for the analysis:

```powershell
kubectl -n demo run api-load --rm -it --image=curlimages/curl --restart=Never -- sh -c "while true; do curl -s -o /dev/null http://api.demo.svc.cluster.local:9090/; sleep 0.2; done"
```

Good version: change `VERSION` in `k8s-api/api.yaml`, keep `ERROR_RATE` at `0`, then commit and push. Argo CD syncs the change and the Rollout reaches 100%.

```powershell
kubectl argo rollouts -n demo get rollout api --watch
```

Bad version: change `VERSION`, set `ERROR_RATE` to `1`, then commit and push. The canary analysis should fail because the success rate drops below 99%; Argo Rollouts marks the rollout aborted and keeps the previous stable version serving.

```powershell
kubectl argo rollouts -n demo get rollout api --watch
kubectl -n demo get analysisrun
```

Rollback through Git:

```powershell
git revert HEAD
git push
```

With Argo CD already watching `main`, the revert should sync and restore the previous version within a few minutes.

## SLO and alert

The SLO is `api:request_success_rate:5m >= 0.99`. Prometheus records it from `flask_http_request_total` and the alert `ApiSloErrorBudgetBurn` fires when the success rate stays below 99% for 1 minute.

Useful checks:

```powershell
kubectl -n demo get servicemonitor api
kubectl -n demo get prometheusrule api-slo
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

PromQL:

```promql
api:request_success_rate:5m
ALERTS{alertname="ApiSloErrorBudgetBurn",alertstate="firing"}
```

## Email alert setup

`argocd/apps/kube-prometheus-stack.yaml` routes alerts to `0973018988a@gmail.com` through Gmail SMTP. Use a Gmail app password, not the normal Gmail account password. Do not commit a real shared password to a public repository; for a real deployment, move it to a Kubernetes Secret or an external secret manager.

Runtime checks:

```powershell
kubectl -n monitoring get pods -l app.kubernetes.io/name=alertmanager
kubectl -n monitoring logs statefulset/alertmanager-kube-prometheus-stack-alertmanager
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
```

Then open Alertmanager at `http://localhost:9093` and confirm `ApiSloErrorBudgetBurn` is routed to `personal-email`.
