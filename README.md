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

You should see a Pod with a single Netifi Broker node deployed in it.

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

In that same shell, build the `ping` and `pong` docker containers:

```bash
./gradlew dockerBuildImage
```

Find your minikube IP address so that you can properly deploy the containers to point to the IP address of the Netifi Broker:

```bash
minikube ip
# 192.168.64.8
```

Start a Pong service that connects statically to the Netifi Broker over the TCP connection, be sure the change
the `ADDRESSES` value with the address of your minikube cluster:

```bash
kubectl run pong1 --image=netifi/pinger-pong --image-pull-policy=Never \
--env="NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=tcp" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=192.168.64.8" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8001"
```

Now start Ping service that connects statically to the Netifi Broker over the TCP connection, and again be
sure to replace the `ADDRESSES` value with the address of your minikube cluster:

```bash
kubectl run ping1 --image=netifi/pinger-ping --image-pull-policy=Never \
--env="NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=tcp" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=192.168.64.8" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8001"
```

You can run:

```bash
minikube dashboard
```

to find the deployments and read the logs, or you can view
the ping and pong endpoints by doing this:

```bash
kubectl port-forward deployment/pong1 8080
open http://localhost:8080/pong
```

or

```bash
kubectl port-forward deployment/ping1 8081
open http://localhost:8081/ping
```

By refreshing the pages, roughly every second, you should see the total counters go up.

Now let's get the name of the service account this tutorial created so that we can use it to
dynamically discover the Netifi Broker our container should connect to:

```bash
kubectl get serviceaccounts
# NAME                       SECRETS   AGE
# default                    1         7m13s
# odd-dog-netifi-broker   1         5m10s
```

Now let's launch another Ping service using our Kubernetes service discovery mechanism.
Be sure to substitute the `serviceaccount` and `DEPLOYMENTNAME` values with your service account name:

```bash
kubectl run ping2 --image=netifi/pinger-ping --image-pull-policy=Never --serviceaccount=odd-dog-netifi-broker \
--env="NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=kubernetes" \
--env="NETIFI_CLIENT_DISCOVERY_KUBERNETESPROPERTIES_CONNECTIONTYPE=tcp" \
--env="NETIFI_CLIENT_DISCOVERY_KUBERNETESPROPERTIES_DEPLOYMENTNAME=odd-dog-netifi-broker"
```

You should see that the Pong service is now incrementing roughly twice as fast, while the Ping
services are maintaining their individual rates.

Let's build these same Docker images on our local computer:

```bash
eval $(minikube docker-env -u)
./gradlew dockerBuildImage
```

And now start an additional Pong service:

```bash
docker run --rm -p 8080:8080 \
-e NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=ws \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=192.168.64.8 \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8101 \
netifi/pinger-pong:latest
```

You can now open: <http://localhost:8080/pong> to see this Pong service taking requests.

Now launch more Ping services:

```bash
for i in {1..5}
do
   docker run -d -P \
   -e NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static \
   -e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=ws \
   -e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=192.168.64.8 \
   -e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8101 \
   netifi/pinger-ping:latest
done
```

You can use `docker ps` to find the ephemeral ports, and launch their respective counter pages in a browser,
or follow their logs, with `docker logs -f <container id>`

When you're done exploring, you can use these commands to remove everything:

```bash
docker rm -f $(docker ps -f "ancestor=netifi/pinger-ping:latest" -q)
docker rm -f $(docker ps -f "ancestor=netifi/pinger-pong:latest" -q)
kubectl delete deployment ping2
kubectl delete deployment ping1
kubectl delete deployment pong1
helm delete odd-dog
minikube delete
```

## Using the Helm chart with GKE

Open Google Cloud Console and build yourself a Kubernetes cluster with 3 default nodes with the
Kubernetes label: key: `type` and value: `broker` with 2 vCPUs each, and a second node pool of 3 nodes with
the Kubernetes label: key: `type` and value: `general` with 2 vCPUs each. To access the Kubernetes labels
you will need to click the 'More options' button and scroll down.

Then get the Kubernetes credentials installed on your local machine via the [Google Cloud SDK](https://cloud.google.com/sdk/install):

```bash
gcloud container clusters get-credentials netifi-demo --zone us-central1-a
```

Install the Tiller RBAC policy:

```bash
kubectl create -f tiller-rbac-config.yaml
```

Initialize Tiller on the cluster

```bash
helm init --service-account tiller --history-max 200
```

And finally setup the and install the Netifi Helm Chart:

```bash
helm repo add netifi https://download.netifi.com/helm-charts/
helm install netifi/netifi-helm-charts -f ./setFiles/gkePublicWS.yaml
```

Now go to the GKE Console, open the Workload tab, and wait for the Netifi Broker cluster to come up.
Now let's find our service account name and launch some Ping containers: 

```bash
kubectl get serviceaccounts
# NAME                          SECRETS   AGE
# default                       1         61m
# rafting-quoll-netifi-broker   1         56m
```

Replace all the references to `rafting-quoll-netifi-broker` in the file: `./setFiles/gkePing.yaml`
with value of the service account you just found.

Now launch the Ping containers:

```bash
kubectl apply -f ./setFiles/gkePing.yaml
# replicaset.apps/ping-rs created
```

You should see 3 Ping containers launch and the logs should show that pings are being generated.
Even though there is no Pong service yet, the requests are being buffered inside the application.

Use Kubectl to get the load balancer EXTERNAL-IP of the Netifi Broker cluster so that we can connect
to it from our laptop:

```bash
kubectl get services
# NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                      AGE
# kubernetes                    ClusterIP      10.52.0.1      <none>          443/TCP                                                                      68m
# rafting-quoll-netifi-broker   LoadBalancer   10.52.15.197   34.67.191.203   8001:32660/TCP,8101:30443/TCP,7001:31100/TCP,6001:30060/TCP,9000:30623/TCP   62m
```

We can now start a Pong service from our laptop. Be sure to change the ADDRESSES value with your EXTERNAL-IP from the previous command:

```bash
docker run --rm -p 8080:8080 \
-e NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=ws \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=34.67.191.203 \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8101 \
netifi/pinger-pong:latest
```

You should see our local Pong service is able to process messages from the Ping services.

Now scale up the Ping services:

```bash
kubectl scale --replicas=91 rs/ping-rs
```

You should see that our local service is still able to keep up with the 91 Ping services.
You'll need to provision more Kubernetes nodes if you want more Ping containers.

Welcome to the Netifi cloud-native application platform!

### Releasing this chart package

```bash
curl -O https://download.netifi.com/helm-charts/index.yaml
helm package .
helm repo index --merge index.yaml .
AWS_PROFILE=netifi-prod  aws s3 cp --acl public-read ./netifi-helm-charts-0.0.3.tgz s3://download.netifi.com/helm-charts/
AWS_PROFILE=netifi-prod  aws s3 cp --acl public-read ./index.yaml s3://download.netifi.com/helm-charts/
# invalidate /helm-charts/index.yaml in cloudfront
```
