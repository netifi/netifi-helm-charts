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
helm repo add netifi https://download.netifi.com/helm-charts/
```

List available chart:

```bash
helm search netifi/
```

## Example Deployments

Local Install:

```bash
helm install netifi/netifi-helm-charts -f ./setFiles/local.yaml
```

Open the minikube dashboard:

```bash
minikube dashboard
```

Get your minikube ip address:

```bash
minikube ip
# 192.168.64.8
```

You use this IP address in your code, and to view the Netifi Broker dashboard:

```bash
open http://$(minikube ip):30004
```

Or open a browser to: <http://192.168.64.8:30004>

In the dashboard you should see 1 broker with a random name. If you click on the name, it will
take you to a broker details page. This page will tell you want information your broker is
advertising to both other brokers, services, and clients.

You'll see that all addresses are currently displaying `172.x.x.x` addresses. This is the address
of the container. Typically this address is only routable from other containers running on the
Kubernetes cluster, and this is ok if all of our applications are running in Kubernetes; however,
let's configure the broker to be accessible natively from our laptop for local development.

Let's delete the existing cluster:

```bash
helm list
# NAME            REVISION        UPDATED                         STATUS          CHART                           APP VERSION     NAMESPACE
# wandering-molly 1               Tue Apr  2 16:16:56 2019        DEPLOYED        netifi-helm-charts-0.0.1        1.6.0           default
```

```bash
 helm delete wandering-molly
# release "wandering-molly" deleted
```

Now let's edit the `local.yaml` file to look like this:

```yaml
netifi-broker:
  enabled: true
  replicaCount: 1
  websocket:
    public:
      addressUsePodIP: false
      address: <MINIKUBE IP ADDRESS>
```

And now let's redeploy:

```bash
helm install netifi/netifi-helm-charts -f ./setFiles/local.yaml
```

Now when we open our dashboard again: <http://192.168.64.8:30004> we should see that the TCP address
is something like: `192.168.64.8:30001`

## Using the Kubernetes Cluster for local development

Go clone our [Pinger](https://github.com/netifi/pinger) example project.

```bash
git clone https://github.com/netifi/pinger.git pinger
cd pinger
```

Configure your shell to use the docker daemon running in minikube:

```bash
eval $(minikube docker-env)
```

Build the `ping` and `pong` docker containers:

```bash
./gradlew dockerBuildImage
```



## Using the Helm chart with GKE

< Insert words and deploying the pinger project on the internet>

### Releasing this chart package

```bash
curl -O https://download.netifi.com/charts/index.yaml
helm package .
helm repo index --merge index.yaml .
# then publish the tgz and index.yaml
```