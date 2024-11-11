## Spin up a k3d environment

```
k3d cluster create --agents 1 rancher-cluster \
  --api-port 6445 \
  --port '8080:80@loadbalancer' \
  --port '8081:443@loadbalancer'
```

### Delete k3d environment

`k3d cluster delete rancher-cluster`

## Install Certmanager

### Docs
https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#4-install-cert-manager

### certmanager releases
https://github.com/cert-manager/cert-manager/releases

### Install certmanager crds

_**Don't do this step if you are running k3d and Traefic (it breaks stuff)**_

If you have installed the CRDs manually, instead of setting `installCRDs` or `crds.enabled` to `true` in your Helm install command, you should upgrade your CRD resources before upgrading the Helm chart:

`kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.crds.yaml`

### Add the Jetstack Helm repository
`helm repo add jetstack https://charts.jetstack.io`

### Update your local Helm chart repository cache
`helm repo update`

### Install the cert-manager Helm chart
```
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
  ```

## Install Rancher

### Docs

https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster

### Add Helm Chart repo

`helm repo add rancher-latest https://releases.rancher.com/server-charts/latest`

### Create namespace for Rancher

`kubectl create namespace cattle-system`

### Show available rancher versions
`helm search repo rancher-latest/rancher --versions`

### Install Rancher

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.my.org \
  --set bootstrapPassword=admin
```

### Check status

`kubectl -n cattle-system rollout status deploy/rancher`

`kubectl -n cattle-system get deploy rancher`

## Access Rancher

Modify /etc/hosts

`127.0.0.1	rancher.my.org`

Navigate to https://rancher.my.org:8081/

Login with `admin` as the bootstrap password

## Install Rancher Istio

### Docs
https://ranchermanager.docs.rancher.com/how-to-guides/advanced-user-guides/istio-setup-guide

`CNI` option documentation: https://ranchermanager.docs.rancher.com/integrations-in-rancher/istio/configuration-options/pod-security-policies

### Install the Istio chart

Navigate to https://rancher.my.org:8081/dashboard/c/local/apps/charts

Install the `Monitoring` chart with the default configuration

Install the `Istio` chart with `Pilot`, `Kiali`, `Ingress Gateway`, `Telemetry`, and `Jaeger Tracing` enabled.  `Kiali` was complaining about a missing `CRD` and took some time before it would install.

## Install a canary OSS Istio Control plane (or use the steps below to use Solo images)

### Docs
https://istio.io/latest/docs/setup/upgrade/canary/

https://istio.io/latest/docs/setup/additional-setup/gateway/#canary-upgrade-advanced 

https://istio.io/latest/blog/2021/revision-tags/

https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-tag

### Check upgrade compatability

`istioctl x precheck`

### Install a new control plane with a revision field

Install a new control plane

`istioctl install --set revision=1-22`

Check to make sure you have two control planes

`kubectl get pods -n istio-system -l app=istiod`

Check for multiple sidecar injector configs

`kubectl get mutatingwebhookconfigurations`

### Create a tag for the new revision

List existing tags

`istioctl tag list`

Create a new tag called `default` for the new control plane

`istioctl tag set default --revision 1-22`

Verify a `MutatingWebhookConfiguration` was created for the tag

`kubectl get MutatingWebhookConfiguration`

## Install a canary Istio control plane using Solo images

### Docs

https://docs.solo.io/gloo-mesh-enterprise/latest/istio/manual/manual_deploy/#control-plane

https://istio.io/latest/blog/2021/revision-tags/

https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-tag

### Install

Use `istio-values.yaml` in this repository as a template values file.  It is currently configured for 1.22.6.

```
helm upgrade --install istiod-1-22 istio/istiod \          
--version 1.22.6 \
--namespace istio-system \
--wait \
-f istio-values.yaml
```

You should see a second `Istiod` instance with the value of `revision:` in `istiod-values.yaml` in the name

`kubectl get pods -n istio-system`

Check for multiple sidecar injector configs

`kubectl get mutatingwebhookconfigurations`

### Create a tag for the new revision

List existing tags

`istioctl tag list`

Create a new tag called `default` for the new control plane

`istioctl tag set default --revision 1-22`

Verify a `MutatingWebhookConfiguration` was created for the tag

`kubectl get MutatingWebhookConfiguration`

## Deploy a sample app

Create test namespace

`kubectl create ns test-ns`

Enable instio sidecar injection

`kubectl label namespace test-ns istio-injection=enabled`

Create a sample pod

`kubectl apply -n test-ns -f sleep/sleep.yaml`

## Cutover to the new control plane

### Cut the `test-ns` namespace over to the new control plane

Remove the `istio-injection` label and add the `istio.io/rev` label

`kubectl label namespace test-ns istio-injection- istio.io/rev=default`

Check the labels

`kubectl get ns test-ns --show-labels`

Restart all pods in the `test-ns` namespace

`kubectl rollout restart deployment -n test-ns`

Check to make sure pods are usng the name control plane

`istioctl proxy-status | grep "\.test-ns"`

## Install Gloo Mesh Core

### Docs

https://docs.solo.io/gloo-mesh-core/latest/setup/install/#single-cluster

### Install

```
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update
```

```
export CLUSTER_NAME=rancher-cluster
export GLOO_VERSION=2.6.6
export GLOO_MESH_CORE_LICENSE_KEY=key
```

Check the key to make sure it is valid

`meshctl license check --key $(echo ${GLOO_MESH_CORE_LICENSE_KEY} | base64 -w0)`

```
helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds \
 --namespace=gloo-mesh \
 --create-namespace \
 --version=$GLOO_VERSION \
 --set installEnterpriseCrds=false
 ```

 Use `gloo-single.yaml` in this repository as a template values file.  It will deploy Gloo Mesh Core with a self-signed TLS server cert.

```
helm upgrade -i gloo-platform gloo-platform/gloo-platform \
  -n gloo-mesh \
  --version $GLOO_VERSION \
  --values gloo-single.yaml \
  --set common.cluster=$CLUSTER_NAME \
  --set licensing.glooMeshCoreLicenseKey=$GLOO_MESH_CORE_LICENSE_KEY
  ```

Open up the Gloo Mesh Core UI

`meshctl dashboard`