# test-argocd

Payment and wallet are internal HTTP echo services for trying Argo CD and reviewing rendered Kubernetes changes. Each has its own Deployment, ClusterIP Service, and generated response ConfigMap in the `demo` namespace.

## Review a change

Open a pull request against `main` that edits an overlay under `apps/payment` or `apps/wallet`. The [reusable semantic diff workflow](https://github.com/charalamposlamprou/kubernetes-manifest-semantic-diff) posts a comment on the PR Conversation tab and updates it on subsequent pushes. Expand a resource in the comment to see its field changes. The inventory in `.manifest-diff.yaml` also covers the Argo CD ApplicationSet. Reviews run without a cluster.

To preview either service locally, run `kubectl kustomize apps/payment` or `kubectl kustomize apps/wallet`.

## Deploy to a demo cluster

With Argo CD and its ApplicationSet controller installed in `argocd`, grant Argo CD read access to this private GitHub repository. Confirm that your current kubectl context points at the intended cluster, then apply the ApplicationSet:

```sh
kubectl apply -k argocd
kubectl -n argocd get applications
```

Argo CD tracks `main`, creates the `demo` namespace, and syncs each service independently. After sync, run `kubectl -n demo port-forward svc/payment-api 8080:80` and visit http://localhost:8080. Use `svc/wallet-api` to try wallet.
