# GKE PV Cluster

```shell
CLUSTER_REGION=us-central1
CLUSTER_ZONE=$CLUSTER_REGION-a
PROJECT_ID=`gcloud config get-value project`

# find appropriate version by
gcloud container get-server-config --zone $CLUSTER_ZONE
CLUSTER_NAME=pv-doom-z
CLUSTER_VERSION=1.11.2-gke.9
```

Create network

```shell
NETWORK=neuron-z
gcloud beta compute networks create --subnet-mode=custom $NETWORK
```

Create subnet

```shell
SUBNET=subneuron-z
SUBNET_RANGE=10.128.0.0/16
gcloud beta compute networks subnets create $SUBNET \
--network $NETWORK \
--range $SUBNET_RANGE \
--secondary-range=containerrange1=192.168.0.0/16 \
--enable-private-ip-google-access \
--region $CLUSTER_REGION
```

Create fw

```shell
gcloud compute firewall-rules create allow-$NETWORK-comms --network $NETWORK --allow tcp:22,tcp:3389,icmp
```

Create cluster

```shell
gcloud beta container clusters create $CLUSTER_NAME \
    --project=$PROJECT_ID \
    --zone=$CLUSTER_ZONE \
    --network=$NETWORK\
    --cluster-version=$CLUSTER_VERSION \
    --enable-ip-alias \
    --create-subnetwork="" \
    --enable-private-nodes \
    --enable-private-endpoint \
    --master-ipv4-cidr=172.16.0.0/28 \
    --enable-stackdriver-kubernetes \
    --addons=HttpLoadBalancing,HorizontalPodAutoscaling,KubernetesDashboard \
    --service-account=gke-node-sa@$PROJECT_ID.iam.gserviceaccount.com
```

```shell
gcloud beta container clusters update $CLUSTER_NAME --enable-master-authorized-networks --master-authorized-networks=$SUBNET_RANGE --zone $CLUSTER_ZONE
```

```shell
OS_IMAGE=`gcloud beta compute images list  --filter="family=debian-9" --format="value(selfLink.scope())"`
OS_PROJECT=debian-cloud
gcloud compute machine-types list --zones=$CLUSTER_ZONE

gcloud beta compute instances create vm-$NETWORK-$SUBNET-jumper-z \
 --zone=$CLUSTER_ZONE \
 --machine-type=f1-micro \
 --subnet=$SUBNET \
 --image=$OS_IMAGE \
 --image-project=$OS_PROJECT \
 --network=$NETWORK \
 --service-account=node-sa@$PROJECT_ID.iam.gserviceaccount.com \
 --scopes=https://www.googleapis.com/auth/cloud-platform \
 --tags=gke-$NETWORK-clients
```

```shell
gcloud beta compute instances create vm-$NETWORK-$SUBNET-tester-z \
 --zone=$CLUSTER_ZONE \
 --machine-type=f1-micro \
 --subnet=$SUBNET \
 --image=$OS_IMAGE \
 --image-project=$OS_PROJECT \
 --network=$NETWORK \
 --service-account=node-sa@$PROJECT_ID.iam.gserviceaccount.com \
 --tags=gke-$NETWORK-clients \
 --scopes=https://www.googleapis.com/auth/cloud-platform \
 --no-address
```

## DEBUG

Get to jumper

```shell
gcloud compute ssh vm-$NETWORK-$SUBNET-jumper-z  --zone $CLUSTER_ZONE
```

From jumper get to tester

```shell
$> gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $PROJECT_ID
$> gcloud compute ssh vm-$NETWORK-$SUBNET-tester-z --internal-ip
```

inside rester debug..

```shell
$>>> sudo apt-get install kubectl
$>>> gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $PROJECT_ID

### verify private IP is being used
$>>> kubectl config view \
-o jsonpath='{.clusters[?(@.name=="'`kubectl config current-context`'")].cluster.server}' | \
xargs curl -I -k {}

curl: (3) [globbing] empty string within braces in column 2
HTTP/2 403
audit-id: df33a83a-4346-4f92-a51f-548dea979dd1
content-type: application/json
x-content-type-options: nosniff
content-length: 269
date: Wed, 17 Oct 2018 03:46:26 GMT

### callout to the api
$>> kubectl get nodes

```

1- confirm peering route with GKE master
2- traceroute from jumper-z to GKE private master
3- traceroute from tester-z to GKE private master
4- confirm google access enabled
5- confirm egress is unblocked