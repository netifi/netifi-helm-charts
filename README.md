# Netifi Helm Charts

## Kubectl

[Install Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

```bash
brew install kubernetes-cli
```

## Minikube

[Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) with [HyperKit](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#hyperkit-driver).

```bash
brew cask install minikube
brew install docker-machine-driver-hyperkit

# docker-machine-driver-hyperkit need root owner and uid
sudo chown root:wheel /usr/local/opt/docker-machine-driver-hyperkit/bin/docker-machine-driver-hyperkit
sudo chmod u+s /usr/local/opt/docker-machine-driver-hyperkit/bin/docker-machine-driver-hyperkit
```

Start Minikube

```bash
minikube start --cpus 4 --memory 8192
```

## Helm

[Install Helm](https://github.com/helm/helm#install) locally on your machine.

```bash
brew install kubernetes-helm
```

[Install](https://helm.sh/docs/using_helm/#role-based-access-control) Tiller Role-based Access Control Service Account

```bash
kubectl create -f tiller-rbac-config.yaml
```

Initialize Tiller on the cluster

```bash
helm init --service-account tiller --history-max 200
```

## Use Chart

Install the repo:

```bash
helm repo add netifi https://downloads.netifi.com/charts/
```

List available chart:

```bash
helm search netifi/
```

## Example Deployments

Local Install:

```bash
helm install netifi/netifi-helm-charts -f ./setFiles/localInternal.yaml
```