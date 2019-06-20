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

You should see a Pod with a single broker node deployed in it.

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

Now start a Pong service that connects statically to the broker over the TCP connection:

```bash
kubectl run pong1 --image=netifi/pinger-pong --image-pull-policy=Never \
--env="NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=tcp" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=192.168.64.8" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8001"
```

And now start Ping service that connects statically to the broker over the TCP connection:

```bash
kubectl run ping1 --image=netifi/pinger-ping --image-pull-policy=Never \
--env="NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=tcp" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=192.168.64.8" \
--env="NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8001"
```

You can use `minikube dashboard` to find the deployments and read the logs, or you can view
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

Roughly every second you should see the counters go up.

Now let's get your service account so that we can dynamically find our brokers:

```bash
kubectl get serviceaccounts
# NAME                       SECRETS   AGE
# default                    1         7m13s
# odd-dog-netifi-broker   1         5m10s
```

This service account is also used by the brokers to discover themselves.

Now let's launch another ping service, only this time let's use Kubernetes service discovery:

```bash
kubectl run ping2 --image=netifi/pinger-ping --image-pull-policy=Never --serviceaccount=odd-dog-netifi-broker \
--env="NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=kubernetes" \
--env="NETIFI_CLIENT_DISCOVERY_KUBERNETESPROPERTIES_CONNECTIONTYPE=tcp" \
--env="NETIFI_CLIENT_DISCOVERY_KUBERNETESPROPERTIES_DEPLOYMENTNAME=odd-dog-netifi-broker"
```

You should see that Pong is now incrementing roughly twice as fast, while the Ping services are
maintaining their individual rates.

Now let's build these same images on our laptop against a locally running Docker daemon:

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

Now launch a bunch more ping services:

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

You can use `docker ps` to find the ephemeral ports, and launch their respective ports, or follow
their logs, with `docker logs -f <container id>`

You can also reopen the Netifi Broker Console: <http://192.168.64.8:9000> select the Groups tab, and
see that there are now 7 destinations in the `com.netifi.pinger.ping` group, and 2 destinations
in the `com.netifi.pinger.pong` group.

If you add or remove more ping or pong services, you should see these values dynamically update
in your browser.

Use these commands to remove everything:

```bash
docker rm -f $(docker ps -f "ancestor=netifi/pinger-ping:latest" -q)
docker rm -f $(docker ps -f "ancestor=netifi/pinger-pong:latest" -q)
kubectl delete deployment ping2
kubectl delete deployment ping1
kubectl delete deployment pong1
helm delete odd-dog
```

## Using the Helm chart with GKE

Open Google Cloud Console and build yourself a Kubernetes cluster with 3 default nodes labeled
with `type: broker` with 2 CPUs each, and another set of 3 nodes with `type: general`.

Then get the credentials for the node via the Google Cloud CLI:

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

```bash
helm repo add netifi https://download.netifi.com/helm-charts/
helm install netifi/netifi-helm-charts -f ./setFiles/gkePublicWS.yaml
```

Go the the GKE Console, find the Daemon Set pods, and wait for them to come up. After it's up you
can take any of the cluster's public IP addresses and go to port 9000 of it to view the Console.

Similarly you can use Kubectl to get the load balancer IP and access it that way:

```bash
kubectl get services
# NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                      AGE
# anxious-seastar-netifi-broker   LoadBalancer   10.7.254.117   35.222.72.248   8001:31887/TCP,8101:30973/TCP,7001:31197/TCP,6001:31033/TCP,9000:31524/TCP   6m
# kubernetes                      ClusterIP      10.7.240.1     <none>          443/TCP                                                                      18m
open http://35.222.72.248:9000
```

From here you can use Docker on your local computer to connect Pong and Ping services to the cluster,
or you can work on building your own Helm Chart to deploy Pong and Ping to the cluster and use
the Kubernetes Discovery plugins to connect to the brokers dynamically.

Here we can even us the load balancer address to connect to the cluster:

```bash
docker run --rm -p 8080:8080 \
-e NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=ws \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=35.222.72.248 \
-e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8101 \
netifi/pinger-pong:latest
```

Now launch a bunch more ping services:

```bash
for i in {1..5}
do
   docker run -d -P \
   -e NETIFI_CLIENT_DISCOVERY_ENVIRONMENT=static \
   -e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_CONNECTIONTYPE=ws \
   -e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_ADDRESSES=35.222.72.248 \
   -e NETIFI_CLIENT_DISCOVERY_STATICPROPERTIES_PORT=8101 \
   netifi/pinger-ping:latest
done
```

And this works because once the initial connection is made the Proteus Client will then use
information from the brokers to establish more connections to the publicly advertised websockets.

### Releasing this chart package

```bash
curl -O https://download.netifi.com/helm-charts/index.yaml
helm package .
helm repo index --merge index.yaml .
# then publish the tgz and index.yaml make sure to make the artifacts public in S3 and invalidate /helm-charts/index.yaml in cloudfront
```