## Overview

This provides  documentation on how to 

* Deploy Rancher on k3d
* Deploy Istio through Rancher
* Deploy a canary Istio control plane using hardened and supported Solo.io images
* View both Istio control planes through Gloo Mesh Core UI
* Cutover to the new control plane

## Prerequisites

* [K3d](https://k3d.io/)
* [Istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/)
* [Meshctl](https://docs.solo.io/gloo-mesh-gateway/main/setup/prepare/meshctl_cli_install/)
* A Gloo Mesh Core [license key](https://docs.solo.io/gloo-mesh-core/2.5.x/setup/prepare/licensing/#get-a-license-key)
* [jq](https://jqlang.github.io/jq/)
* [Go](https://go.dev/doc/install)
* [Kubectl-grep](https://github.com/howardjohn/kubectl-grep)


## Spin up a k3d environment

```
k3d cluster create --agents 1 rancher-cluster \
  --api-port 6445 \
  --port '8080:80@loadbalancer' \
  --port '8081:443@loadbalancer' \
  --port '8443:8443@loadbalancer' \
  --k3s-arg="--disable=traefik@server:0"
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

### Show available rancher versions
`helm search repo rancher-latest/rancher --versions`

### Install Rancher

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.my.org \
  --set bootstrapPassword=admin \
  --set ingress.enabled=false
```

### Check status

`kubectl -n cattle-system rollout status deploy/rancher`

`kubectl -n cattle-system get deploy rancher`

### Expose the Rancher deployment

`kubectl patch svc rancher -n cattle-system -p '{"spec": {"type": "LoadBalancer"}}'`

`kubectl patch svc rancher -n cattle-system --type merge -p '{"spec":{"ports": [{"port": 8443,"name":"http","targetPort": 443}]}}'`

## Access Rancher

Modify /etc/hosts

```
127.0.0.1 rancher.my.org
127.0.0.1 productpage.my.org
```

Navigate to https://rancher.my.org:8443/

Login with `admin` as the bootstrap password

## Install Rancher Istio

### Docs
https://ranchermanager.docs.rancher.com/how-to-guides/advanced-user-guides/istio-setup-guide

`CNI` option documentation: https://ranchermanager.docs.rancher.com/integrations-in-rancher/istio/configuration-options/pod-security-policies

### Install the Istio chart

Navigate to https://rancher.my.org:8443/dashboard/c/local/apps/charts

Install the `Monitoring` chart with the default configuration

Install the `Istio` chart with `Pilot`, `Kiali`, `Ingress Gateway`, `Telemetry`, and `Jaeger Tracing` enabled.  `Kiali` was complaining about a missing `CRD` and took some time before it would install.

## Expose the Rancher Istio ingress gateway

`kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'`

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

## Deploy a gateway using Solo images

### Docs

https://github.com/istio/istio/tree/release-1.24/manifests/charts/gateways/istio-ingress

https://istio.io/latest/docs/setup/additional-setup/gateway/

### Install

Use `gateway-values.yaml` in this repository as a template values file.  It is currently configured for 1.22.6.  Replace `global.hub` and `service.loadBalancerIP` with actual values.

```
helm upgrade --install istio-gateway istio/gateway \
--version 1.22.6 \
--namespace istio-system \
--wait \
-f gateway-values.yaml
```

This will provision a second gateway attached to the new `1-22` Istio control plane as detailed [here](https://istio.io/latest/docs/setup/additional-setup/gateway/#canary-upgrade-advanced).

## Deploy a sample app and configure routing

Enable instio sidecar injection

`kubectl label namespace default istio-injection=enabled`

Deploy a modified version of the [bookinfo](https://github.com/istio/istio/tree/master/samples/bookinfo) application

`kubectl apply -f bookinfo.yaml`

Configure bookinfo for routing using the Rancher ingress gateway

`kubectl apply -f bookinfo-routing.yaml`

Confirm that all 4 pods in the default namespace are connected to the default Rancher mesh

`istioctl proxy-status | grep "\.default"`

Confirm you can hit the deployed application using the Rancher Istio route by navigating to `http://productpage.my.org:8080/productpage`

Create a `port-forward` to the second gateway instance to test canary routing

`kubectl port-forward -n istio-system deploy/istio-ingressgateway-canary 8080:80`

Ensure you can hit the application on the canary route by navigating to `http://localhost:8080`

## Create a tag for the new revision

### Docs

https://istio.io/latest/blog/2021/revision-tags/

https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-tag

### Create a revision tag

List existing tags

`istioctl tag list`

Create a new tag called `canary` for the new control plane

`istioctl tag set canary --revision 1-22 --overwrite`

Verify a `MutatingWebhookConfiguration` was created for the tag

`kubectl get MutatingWebhookConfiguration`

## Check application cert chain on the Rancher deployed Istio

`kubectl exec -t  deploy/reviews-v1 -- openssl s_client -showcerts -connect ratings:9080 -alpn istio`

Under `1 s:O = cluster.local`, note the certificate between `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----`

## Cutover Overview

![Cutover](/images/Cutover.png)

## Cut the application over to the new control plane

### Cut the `default` namespace over to the new control plane

Remove the `istio-injection` label and add the `istio.io/rev` label

`kubectl label namespace default istio-injection- istio.io/rev=canary`

Check the namespace labels

`kubectl get ns default -L istio.io/rev`

Restart all pods in the `default` namespace

`kubectl rollout restart deployment`

Check to make sure the pods in the default namespace are usng the canary control plane.  You should see `1.22.6-solo`

`istioctl proxy-status | grep "\.default"`

## Check application cert chain on the canary Istio

Make sure the root CA is the same as previously so that everything is trusted.

`kubectl exec -t  deploy/reviews-v1 -- openssl s_client -showcerts -connect ratings:9080 -alpn istio`

Under `1 s:O = cluster.local`, note the certificate between `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----`.  Compare it to the previous root certificate that was used for Rancher deploy Istio and confirm they are the same.

## Cut routing over to the new control plane, scale Rancher Istio Gateway to 0, and test

Configure the service to point to the canary gateway

`kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"selector":{"app": "istio-ingressgateway-canary", "istio": "istio-ingressgateway-canary"}}}'`

Scale the Rancher Istio gateway to zero instances

`kubectl scale deployment istio-ingressgateway --replicas=0 -n istio-system`

Confirm the Rancher Istio Gateway is at zero instances

`kubectl get deploy istio-ingressgateway -n istio-system`

Confirm you can hit the deployed application using the canary Istio route by navigating to `http://productpage.my.org:8080/productpage`

## Mark the Rancher deployed Istio CRDs as Helm managed install the istio-base Helm chart

Apply the following function per [this](https://github.com/helm/helm/issues/2730#issuecomment-2128275312) Github issue:

```
function helm-install-takeover() {                           
  release=$1
  release_namespace=$2
  shift
  shift
  while read -r line; do
    kind=`<<<$line cut -d/ -f1`
    nokind=`<<<$line cut -d/ -f2`
    namespace=`<<<${nokind} rev |cut -d. -f1|rev`
    name=${nokind%$namespace}
    name="${name%.}"
    kubectl annotate $kind $name ${namespace:+--namespace=$namespace} meta.helm.sh/release-name=$release
    kubectl annotate $kind $name ${namespace:+--namespace=$namespace} meta.helm.sh/release-namespace=$release_namespace
    kubectl label $kind $name ${namespace:+--namespace=$namespace} app.kubernetes.io/managed-by=Helm
  done <<< $(helm template $release -n $release_namespace $@ | kubectl grep -s)
  helm install $release -n $release_namespace $@
}
```

Run the following command to label/annotate all Rancher deployed Istio Custom Resource Definitions (CRDs) in the `istio-system` namespace as Helm managed.  This requires the [Kubectl-grep](https://github.com/howardjohn/kubectl-grep) `kubectl` plugin.  You can confirm this plugin is available by running `kubectl plugin list` and noting if `kubectl-grep` is available.  This will allow us to manage these resources with the [istio-base](https://github.com/istio/istio/tree/master/manifests/charts/base) Helm chart.

`helm-install-takeover istio-base istio-system istio/base`

You should see the following output:

```
helm-install-takeover istio-base istio-system istio/base                     
customresourcedefinition.apiextensions.k8s.io/wasmplugins.extensions.istio.io annotated
customresourcedefinition.apiextensions.k8s.io/wasmplugins.extensions.istio.io annotated
customresourcedefinition.apiextensions.k8s.io/wasmplugins.extensions.istio.io labeled
...
```

Install the `istio-base` helm chart to manage these CRDs:

`helm upgrade --install istio-base istio/base -n istio-system --set defaultRevision=1-22`

You should see the following output:

```
helm upgrade --install istio-base istio/base -n istio-system --set defaultRevision=1-22
Release "istio-base" has been upgraded. Happy Helming!
NAME: istio-base
LAST DEPLOYED: timestamp
NAMESPACE: istio-system
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
Istio base successfully installed!
```

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

Install the Gloo Mesh Core CRDs

```
helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds \
 --namespace=gloo-mesh \
 --create-namespace \
 --version=$GLOO_VERSION \
 --set installEnterpriseCrds=false \
 --set featureGates.insightsConfiguration=true
 ```

 Apply the `insightsconfig.yaml` file to work around [this](https://github.com/solo-io/gloo-mesh-enterprise/issues/14763) bug in the Gloo UI when using Istio revision tags.

 `kubectl create -f insightsconfig.yaml`

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
