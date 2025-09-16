# k0rdent-utils
A guide to creating and modifying customer k0rdent templates

Table of Contents

* [Overview](#overview)
* [Create your own Templates](#create-your-own-tenplates)
* [Adding your repo to k0rdent](#add-the-repo-to-k0rdent)
* [Checking your work](#checking-your-work)

## Overview

This is a rough guide on the process of creating your own custom k0rdent Cluster Templates. 

The basic process shown here requires you to have access to GitHub and have the actions 
setup to turn your charts into packages that can be served from github. An example of this 
can be found in the [k0rdent-utils](https://github.com/p5ntangle/k0rdent-utils) project
on github.

Essential Skills:

* Working knowledge of Helm Charts
* Basic git commit skills

Local tools:

* Git
* Helm
* Kubectl (or Lens)

## Create your own Tenplates

To create your own templates you need to have a basic understanding of the CAPI 
providers CRD model `infrastructure.cluster.x-k8s.io` and `cluster.x-k8s.io`.

There are at least 7 objects that make up a typical Cluster Template (this is a 
example for AWS):

| Typical Filename | apiVersion | Object | Purpose |
|-----------------| ------------|--------| --------|
| awscluster.yaml | apiVersion: infrastructure.cluster.x-k8s.io/v1beta2 | kind: AWSCluster | Common Cluster Config |
| awsmachinetemplate-controlplane.yaml | apiVersion: infrastructure.cluster.x-k8s.io/v1beta2 | kind: AWSMachineTemplate | Control Plane Machine config $^{1}$ |
| awsmachinetemplate-worker.yaml | apiVersion: infrastructure.cluster.x-k8s.io/v1beta2 | kind: AWSMachineTemplate | Worker node Machine config $^{2}$ |
| cluster.yaml | apiVersion: cluster.x-k8s.io/v1beta1 | kind: Cluster | CAPI Cluster definition reference |
| k0scontrolplane.yaml | apiVersion: controlplane.cluster.x-k8s.io/v1beta1 | kind: K0sControlPlane | k0s control plane config |
| k0sworkerconfigtemplate.yaml | apiVersion: bootstrap.cluster.x-k8s.io/v1beta1 | kind: K0sWorkerConfigTemplate | k0s worker config |
| machinedeployment.yaml | apiVersion: cluster.x-k8s.io/v1beta1 | kind: MachineDeployment | Machine deployment config |

1. Typical there should only be one control plane temples
2. There could be multiple work templates, if you want to have multiple machine types in a cluster

Once you have made your changes and setup you configuration there are few commands that
can help you validate your work.

Start by creating a version of the [`Cluster Deployment`](https://docs.k0rdent.io/latest/user/user-create-cluster/) yaml locally removing everything above and including the `spec` and saving it as `testvalues.yaml` 

### Checking your work

1. Lint the chart locally

``` BASH
helm lint ./mychart
```

2. Render the chart to see the final YAML (assumes you are in your charts root folder).

``` BASH
helm template myrelease ./ --values testvalues.yaml
```

3. Test the ouput by providing a example cluster deployment against a live system

``` BASH
helm install  my-charts-name ./ --dry-run --values testvalues.yaml --debug
```

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

2. Create the template object

NOTE: the object is immutable, so if you need to change something you need to delete is and recreate it, don't do this after deploying a cluster with the template (see template chains for that)

``` YAML
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterTemplate
metadata:
  labels:
    k0rdent.mirantis.com/component: kcm
  name: aws-standalone-cp-security
  namespace: kcm-system
spec:
  helm:
    chartSpec:
      chart: aws-standalone-cp-security
      interval: 10m0s
      reconcileStrategy: Revision
      sourceRef:
        kind: HelmRepository
        name: p5ntangle-templates
      version: 0.0.1
```

Simple command to add it from the command line.

``` BASH
cat <<EOF | kubectl apply -f -
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterTemplate
metadata:
  labels:
    k0rdent.mirantis.com/component: kcm
  name: aws-standalone-cp-security
  namespace: kcm-system
spec:
  helm:
    chartSpec:
      chart: aws-standalone-cp-security
      interval: 10m0s
      reconcileStrategy: Revision
      sourceRef:
        kind: HelmRepository
        name: p5ntangle-templates
      version: 0.0.1
EOF
```

3. Check the status of the chart by examining the crd object.

`kubectl get clustertemplates aws-standalone-cp-security -n kcm-system -o yaml`

## Useful Template Modifications

### Extra Ingress Rules

#### AWS 
The AWS Provider supports adding extra ingress rules in the network section

``` YAML
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
spec:
    ~~
    network:
        AdditionalNodeIngressRules:
        - description: grpc access rules for network automation
          fromPort: 5111
          protocol: tcp
          toPort: 5111
```

in progress

``` YAML
  
    AdditionalNodeIngressRules:
    {{- range $i, $value := .Values.worker.AdditionalNodeIngressRules }}
      - description: {{ $value }}
    {{- range $key, $val := $value }}
        {{ $key }}: {{ $val }}
    {{- end }}
```

## Github Workflow Notes

1. Ensure that Chart.yaml and the workflow have the same version number

## Secure registry Authentication

THIS NEEDS TO BE TESTED

Here are the common ways to authenticate a Flux HelmRepository (HTTP/S and OCI). Pick the one that matches your repo.

1) Basic auth (username/password) — HTTP/S

Create a Secret with username and password, then reference it in the HelmRepository.

``` BASH
kubectl -n default create secret generic repo-creds \
  --from-literal=username='my-user' \
  --from-literal=password='my-pass'
```

``` YAML
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-repo
  namespace: default
spec:
  interval: 5m
  url: https://charts.example.com
  secretRef:
    name: repo-creds
```

Flux expects .data.username and .data.password for basic auth.  ￼

2) Client TLS / custom CA — HTTP/S

If your repo requires a client certificate or uses a private CA, put the PEMs in a Secret and point certSecretRef at it.

``` BASH
# tls.crt + tls.key optional pair, and/or ca.crt
flux create secret tls repo-tls \
  --tls-crt-file=client.crt \
  --tls-key-file=client.key \
  --ca-crt-file=ca.crt \
  --namespace=default
```

``` YAML
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-repo
  namespace: default
spec:
  interval: 5m
  url: https://charts.example.com
  certSecretRef:
    name: repo-tls
```

Use tls.crt/tls.key for client auth and ca.crt for server verification. (TLS via certSecretRef is the current, non-deprecated way.)  ￼

3) OCI-backed Helm repo

Set type: oci. You can use basic auth or a dockerconfigjson-style pull secret.

Option A — username/password:

``` YAML
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-oci
  namespace: default
spec:
  type: oci
  url: oci://ghcr.io/my-org/charts
  secretRef:
    name: oci-basic
---
apiVersion: v1
kind: Secret
metadata:
  name: oci-basic
  namespace: default
stringData:
  username: my-user
  password: my-token
```

Option B — docker-registry secret (recommended for registries):

``` BASH
kubectl -n default create secret docker-registry ghcr-auth \
  --docker-server=ghcr.io \
  --docker-username=flux \
  --docker-password="$GITHUB_PAT"
```

``` YAML
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-oci
  namespace: default
spec:
  type: oci
  url: oci://ghcr.io/my-org/charts
  secretRef:
    name: ghcr-auth
```

OCI HelmRepositories accept dockerconfigjson secrets; Flux also supports cloud-provider auth via .spec.provider (aws, azure, gcp) when pulling from ECR/ACR/GCR.  ￼
