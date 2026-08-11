# k8s-helm-service-deploy

A production-shaped Helm chart for deploying a containerized HTTP service to
Kubernetes — Deployment, Service, HorizontalPodAutoscaler, ConfigMap,
PodDisruptionBudget, and an optional Ingress — with the defaults set the way
I'd actually want them running, not the bare minimum that happens to work.

Built as the Kubernetes counterpart to
[`cicd-pipeline-aws-ecs`](https://github.com/Param-Cloudtelecom/cicd-pipeline-aws-ecs):
same container image, two different orchestrators, so the trade-offs between
"ECS Fargate behind an ALB" and "Kubernetes Deployment behind an Ingress"
are directly comparable side by side.

## What's in the chart

| Template | Purpose |
|---|---|
| `deployment.yaml` | Rolling update (`maxUnavailable: 0`) with liveness/readiness probes, resource requests/limits, and a locked-down `securityContext` |
| `service.yaml` | ClusterIP fronting the pods |
| `hpa.yaml` | Target-tracking autoscaling on CPU **and** memory |
| `pdb.yaml` | PodDisruptionBudget so a node drain/upgrade can't take the whole service down at once |
| `configmap.yaml` | Non-secret env vars, mounted via `envFrom` |
| `serviceaccount.yaml` | Dedicated ServiceAccount (not `default`) per release |
| `ingress.yaml` | Optional, off by default — enable per environment |

## Security defaults that are actually on by default

- `runAsNonRoot: true` at the pod level, matching the `USER appuser` (uid
  10001) baked into the Dockerfile in `cicd-pipeline-aws-ecs`
- `readOnlyRootFilesystem: true` — the only writable path is an `emptyDir`
  mounted at `/tmp`
- `allowPrivilegeEscalation: false` and `capabilities.drop: [ALL]`
- No `imagePullSecrets` or cluster-admin-anything baked into the chart

## Usage

```bash
helm lint charts/service
helm template demo-app charts/service -f environments/values-dev.yaml

# install/upgrade against a real cluster
helm upgrade --install demo-app charts/service \
  -f environments/values-dev.yaml \
  --namespace demo-app --create-namespace
```

`environments/values-dev.yaml` and `environments/values-prod.yaml` show the
same chart tuned two different ways — single replica / no autoscaling / no
TLS in dev, versus 3+ replicas / autoscaling to 12 / cert-manager-issued TLS
/ pod anti-affinity spreading replicas across nodes in prod. The base
`charts/service/values.yaml` defaults sit in between as a sane starting
point.

## CI

`.github/workflows/lint-chart.yml` runs `helm lint`, renders the chart with
all three value sets (`helm template`), and validates the rendered output
against the upstream Kubernetes OpenAPI schema with
[`kubeconform`](https://github.com/yannh/kubeconform) — so a template typo
or an invalid field name fails the pipeline before it ever reaches a
cluster, not after.

## Related

Deploys the same image built and pushed by
[`cicd-pipeline-aws-ecs`](https://github.com/Param-Cloudtelecom/cicd-pipeline-aws-ecs).
For the equivalent stack on plain AWS (ECS Fargate + ALB + RDS, no
Kubernetes control plane to run), see
[`terraform-aws-infra-modules`](https://github.com/Param-Cloudtelecom/terraform-aws-infra-modules).

## License

MIT — see [LICENSE](LICENSE).
