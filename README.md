# test-argocd

Payment and wallet are internal HTTP echo services managed with Argo CD. Each service has a Kustomize base and three deployable environment overlays. Each overlay renders a Deployment, ClusterIP Service, and generated response ConfigMap.

| Environment | Kubernetes namespace | Payment replicas | Wallet replicas |
| --- | --- | ---: | ---: |
| dev | `demo` | 2 | 1 |
| staging | `demo-staging` | 3 | 3 |
| prod | `demo-prod` | 4 | 3 |

All three environments use namespaces on the same demo cluster. In prod, each service has a PodDisruptionBudget allowing one unavailable pod during voluntary disruptions.

## Where to make a change

| Path | Scope |
| --- | --- |
| `apps/<service>/overlays/<environment>/kustomization.yaml` | One service in one environment. Set replicas here, or add an image override or configuration patch. |
| `apps/<service>/base/kustomization.yaml` | Service identity and response configuration across its environments. |
| `common/http-echo/` | Workload and Service defaults shared by both echo services. |
| `argocd/applicationset.yaml` | Application generation, source paths, destination namespaces, and sync policy. |
| `.manifest-diff.yaml` | The six overlays and the Argo CD bootstrap reviewed on pull requests. |

For example, edit the replica count in `apps/payment/overlays/staging/kustomization.yaml` and run:

```sh
kubectl kustomize apps/payment/overlays/staging
```

Open a PR against `main`. The [reusable semantic diff workflow](https://github.com/charalamposlamprou/kubernetes-manifest-semantic-diff) renders the base and head revisions and posts a comment grouped by service and environment. A staging-only replica change appears under payment / staging. To promote it, make the corresponding edit in the prod overlay in a subsequent PR.

## Deploy to a demo cluster

With Argo CD and its ApplicationSet controller installed in `argocd`, grant Argo CD read access to this private GitHub repository. Once the overlays are on `main`, apply the bootstrap to the intended cluster:

```sh
kubectl apply -k argocd
kubectl -n argocd get applications -l environment
```

The ApplicationSet discovers service bases under `apps/*/base` and creates one Application per service and environment. Each Application tracks `main`, creates its namespace, and syncs automatically with pruning and self-healing enabled. For an existing installation, reapply the bootstrap to enable staging and prod. The original `apps/payment` and `apps/wallet` entry points forward to dev, preserving the existing development workloads during the transition.

After sync, run `kubectl -n demo port-forward svc/payment-api 8080:80` and visit http://localhost:8080. Use `svc/wallet-api` for wallet or change the namespace to try a different environment.
