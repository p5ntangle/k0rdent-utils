# Updating or adding a Provider

This is a work in progress.

## Add the repo to k0rdent

Deploying a custom template to k0rdent requires a few basic setup steps:

* Setup a HelmRepository link to your templates repository
* Create the template object in k0rdent
* Validate that the template is working

The example below is using a open GitHub OCI registry.

1. In order to use the templates you will need to add the helm repo that has you helm chart in it to k0rdent.

``` YAML
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  labels:
    k0rdent.mirantis.com/managed: 'true'
  name: p5ntangle-templates
  namespace: kcm-system
spec:
  interval: 10m0s
  provider: generic
  type: oci
  url: oci://ghcr.io/p5ntangle/k0rdent-templates
```

Simple command to add it from the command line.

``` BASH
cat <<EOF | kubectl apply -f -
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  labels:
    k0rdent.mirantis.com/managed: 'true'
  name: p5ntangle-templates
  namespace: kcm-system
spec:
  interval: 10m0s
  provider: generic
  type: oci
  url: oci://ghcr.io/p5ntangle/k0rdent-templates
EOF
```

2. Create the Provider Template object

>[!NOTE]
> the object is immutable, so if you need to change something you need to delete is and recreate it, don't do this after deploying a cluster with the template (see template chains for that)

>[!WARNING]
>The `ownerReferences` usage here is a hack, your mileage may vary, you have been warned.

``` YAML
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ProviderTemplate
metadata:
  labels:
    k0rdent.mirantis.com/component: kcm
  name: cluster-api-provider-aws-1-0-5
  ownerReferences:
    - apiVersion: k0rdent.mirantis.com/v1beta1
      kind: Release
      name: kcm-1-3-1
      uid: 2af0eb23-6fb6-4d09-9fef-a54105e5edb9
  namespace: kcm-system
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-aws
      interval: 10m0s
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: p5ntangle-templates
      version: 1.0.5
```



Simple command to add it from the command line.

``` BASH
cat <<EOF | kubectl apply -f -
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ProviderTemplate
metadata:
  labels:
    k0rdent.mirantis.com/component: kcm
  name: cluster-api-provider-aws-som
  ownerReferences:
    - apiVersion: k0rdent.mirantis.com/v1beta1
      kind: Release
      name: kcm-1-3-1
      uid: 2af0eb23-6fb6-4d09-9fef-a54105e5edb9
  namespace: kcm-system
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-aws
      interval: 10m0s
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: p5ntangle-templates
      version: 1.0.5
EOF
```

3. Check the status of the chart by examining the crd object.

`kubectl get clustertemplates aws-standalone-cp-security -n kcm-system -o yaml`