# Custom applications
This directory is watched by the `appset-custom.yaml` ApplicationSet and allows generating Applications for any namespace. 
This is useful when the Application you want to create doesn't follow the standard patterns of other ApplicationSets.

The file structure for creating a new custom Application within this directory follows the convention:
`<NAMESPACE>`/`<APPLICATION_NAME>`.

For example, to generate a new Application named "test", in the "foo" namespace, you would create `foo/test/` inside of `/custom`
and place your manifests/Helm chart/Kustomization within the `test` folder.
