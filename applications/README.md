This Helm chart defines the `Applications` and `ApplicationSets` that ArgoCD should manage.

Any new applications that you want to deploy should be defined here under `/templates`. If you need to change values or Kustomize the deployments, you can point the application source at a subfolder of this repository (e.g. `core/argocd`) and make whatever configuration changes you need.

To easily deploy new applications without having to write a new Application manifest, you can configure it in one of the `ApplicationSet` directories. These will automatically generate the `Application` spec from the template, based on the directory you configure it in. 