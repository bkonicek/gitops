# gitops

Repository storing ArgoCD configurations for my personal K8s cluster running in OKE

## Infrastructure
Infrastructure as Code via Terraform is maintained in [bkonicek/terraform](https://github.com/bkonicek/terraform)

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
- If installing on a new cluster, and you want to reuse the same `sealed-secrets` cert and private key, you have to update the `Secret` after sealed-secrets is successfully installed.
    - `kubectl edit secret -n kube-system sealed-secrets-key`
    - Replace `tls.crt` and `tls.key` with the old values from a previous installation
    - Restart the sealed-secrets pod(s) `kubectl rollout restart -n kube-system deployment sealed-secrets-controller`


## Configuring Applications
New applications should be added to the specific cluster's subfolder within the `environments/` directory. Once added they are automatically detected by the `ops-tools` ApplicationSet.

### Structure
Application directories must contain a `config.yaml` file with the following structure:

```yaml 
# Required Fields
namespace: default

# Optional Fields
repoURL:        # defaults to this repo. If set to another repo, must be a valid Helm repository and requires `chartName` to be specified.
targetRevision: # defaults to HEAD, for external charts should be the version
chartName:      # defaults to the name of the directory, required if repoURL is set to an external Helm repository
releaseName:    # defaults to the generated application name
chartPath:      # defaults to "charts/<application directory name>". Ignored if using external Helm repository as `repoURL`.
```

Optionally, each application can have a `values.yaml` file in the same directory to set values for the Helm chart specific to that environment. If you want to set values that apply to all environnments, create a file in `base/` matching the name of the application directory, e.g. `base/metrics-server.yaml`.

#### Custom Charts
If you need to add additional templates to a chart that it doesn't natively support, add a Helm chart folder to `charts/` with the customizations you want, defining the upstream chart as a subchart. The folder name must match the name of the application directory in `environments/`. `chartName` is optional in this case.

You can create a `values.yaml` within the custom chart to set any default values you want all instances of this application to use.

**Example**

[External Secrets Operator Application](environments/sandbox-oci/external-dns)

[External Secrets Operator Custom Chart](charts/external-dns)

#### External Charts
If you only need to set custom values for an upstream chart but otherwise deploy it as it is published, you can set `repoURL` to the chart's Helm repository or git URL. If the former, set `chartName` as the name of the chart. For the latter, set `chartPath` to the path of the chart within the git repository but do not set `chartName`. You can then use `values.yaml` to set any values you want for that chart.

**Example**

[Sealed-Secrets Application](environments/sandbox-oci/sealed-secrets-controller)

[Sealed-Secrets Base Values](base/sealed-secrets-controller.yaml)
