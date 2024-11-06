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

## Install Istio

### Docs
https://ranchermanager.docs.rancher.com/how-to-guides/advanced-user-guides/istio-setup-guide

`CNI` option documentation: https://ranchermanager.docs.rancher.com/integrations-in-rancher/istio/configuration-options/pod-security-policies

### Install the Istio chart

Navigate to https://rancher.my.org:8081/dashboard/c/local/apps/charts

Install the `Monitoring` chart with the default configuration

Install the `Istio` chart with `Pilot`, `Kiali`, `Ingress Gateway`, `Telemetry`, and `Jaeger Tracing` enabled.  `Kiali` was complaining about a missing `CRD` and took some time before it would install.

