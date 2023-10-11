This Helm chart defines the `Applications` that ArgoCD should manage.

Any new applications that you want to deploy should be defined here under `/templates`. If you need to change values or Kustomize the deployments, you can point the application source at a subfolder of this repository (e.g. `core/argocd`) and make whatever configuration changes you need.