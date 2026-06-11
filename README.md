# GitOps safe API rollout

This repo deploys the `api` service through Argo CD and Argo Rollouts. A new version is promoted only when Prometheus reports at least 99% successful HTTP requests. If the canary returns too many 5xx responses, the Rollout analysis fails and Argo Rollouts aborts back to the previous ReplicaSet.

## What is included

- GitOps entrypoint: `argocd/root.yaml`
- API application: `argocd/apps/api.yaml`
- Rollout and service: `k8s-api/api.yaml`
- Canary gate: `k8s-api/analysis-template.yaml`
- Metrics scrape: `k8s-api/servicemonitor.yaml`
- SLO alert: `k8s-api/prometheus-rule.yaml`
- Monitoring stack: `argocd/apps/kube-prometheus-stack.yaml`

## Demo commands

Check Argo CD sync:

```powershell
kubectl -n argocd get applications
kubectl -n demo get rollout api
kubectl argo rollouts -n demo get rollout api
```

Generate traffic so Prometheus has data for the analysis:

```powershell
kubectl -n demo run api-load --rm -it --image=curlimages/curl --restart=Never -- sh -c "while true; do curl -s -o /dev/null http://api.demo.svc.cluster.local:9090/; sleep 0.2; done"
```

Good version: change `VERSION` in `k8s-api/api.yaml`, keep `ERROR_RATE` at `0`, then commit and push. Argo CD syncs the change and the Rollout reaches 100%.

Bad version: change `VERSION`, set `ERROR_RATE` to `1`, then commit and push. The canary analysis should fail because the success rate drops below 99%; Argo Rollouts marks the rollout aborted and keeps the previous stable version serving.

Rollback through Git:

```powershell
git revert HEAD
git push
```

## Email alert setup

`argocd/apps/kube-prometheus-stack.yaml` routes alerts to `quyetmarcus04@gmail.com`. Replace `CHANGE_ME_GMAIL_APP_PASSWORD` with a Gmail app password before pushing if email delivery must work in the cluster. Do not commit a real shared password to a public repository; for a real deployment, move it to a Kubernetes Secret or an external secret manager.
