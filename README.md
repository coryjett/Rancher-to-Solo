## Spin up k3d environment
```
export MGMT=mgmt
export DEMO_INFRA_DIR=/Users/coryjett/Documents/Workspace/Solo/demo-infrastructure
source $DEMO_INFRA_DIR/k3d/lib.sh
create-k3d-cluster $MGMT $DEMO_INFRA_DIR/k3d/$MGMT.yaml
```

### Delete k3d environment
`delete-k3d-cluster $MGMT`

## Install Rancher docs

https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster

## Add Helm Chart repo

`helm repo add rancher-latest https://releases.rancher.com/server-charts/latest`

## Create namespace for Rancher

`kubectl create namespace cattle-system`

## Install certmanager

### Docs
https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#4-install-cert-manager

### certmanager releases
https://github.com/cert-manager/cert-manager/releases

### If you have installed the CRDs manually, instead of setting `installCRDs` or `crds.enabled` to `true` in your Helm install command, you should upgrade your CRD resources before upgrading the Helm chart:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.crds.yaml

### Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

### Update your local Helm chart repository cache
helm repo update

### Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

## Install Rancher

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