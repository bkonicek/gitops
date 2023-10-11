# gitops

Repository storing ArgoCD configurations for my personal K8s cluster running in OKE

## Bootstrapping

1. Assuming you have a running cluster, install the latest ArgoCD [release](https://github.com/argoproj/argo-cd/releases)

    ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/{VERSION}/manifests/install.yaml
    ```

2. Update https://github.com/bkonicek/gitops/blob/8e3afc7fe45ec8c423f14521e0232dbd93524e6a/core/argocd/kustomization.yaml#L7 with the same release version that you installed in step 1, and commit it to the repo.

3. Apply the root app-of-apps [`Application`](https://github.com/bkonicek/gitops/blob/main/application.yaml)

```
kubectl apply -f application.yaml
```

## Additional Requirements
- If installing on a new cluster, and you want to reuse the same sealed-secrets cert and private key, you have to update the `Secret` after sealed-secrets is successfully installed.
    - `kubectl edit secret -n kube-system sealed-secrets-key`
    - Replace `tls.crt` and `tls.key` with the old values from a previous installation
    - Restart the sealed-secrets pod(s) `kubectl rollout restart -n kube-system deployment sealed-secrets-controller`