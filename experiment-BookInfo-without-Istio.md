# Experiment BookInfo **without** Istio

## Create a Kubernetes cluster of 4 nodes on GCP

Verifiy that your project is set:
```
~ $ gcloud config list
```
```
[component_manager]
disable_update_check = True
[compute]
gce_metadata_read_timeout_sec = 5
[core]
account = patrice.krakow@gmail.com
disable_usage_reporting = False
project = istio-project-01
[metrics]
environment = devshell
```
Let's configure the zone to use the Google data center in Belgium:
```
$ gcloud config set compute/zone europe-west1-b
```
```
Updated property [compute/zone].
~ $ gcloud config list
[component_manager]
disable_update_check = True
[compute]
gce_metadata_read_timeout_sec = 5
zone = europe-west1-b
[core]
account = patrice.krakow@gmail.com
disable_usage_reporting = False
project = istio-project-01
[metrics]
environment = devshell
```
Create a new cluster:
```
~ $ gcloud container clusters create istio-cluster-01 \
  --num-nodes=4
```
```
WARNING: Starting in 1.12, new clusters will have basic authentication disabled by default. Basic authentication can be enabled (or disabled) manually using the `-
-[no-]enable-basic-auth` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate usin
g the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-ena
ble-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cl
uster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorep
airs.
WARNING: Starting in Kubernetes v1.10, new clusters will no longer get compute-rw and storage-ro scopes added to what is specified in --scopes (though the latter w
ill remain included in the default --scopes). To use these scopes, add them explicitly to --scopes. To use the new behavior, set container/new_scopes_behavior prop
erty (gcloud config set container/new_scopes_behavior true).
Creating cluster istio-cluster-01 in europe-west1-b...done.
Created [https://container.googleapis.com/v1/projects/istio-project-01/zones/europe-west1-b/clusters/istio-cluster-01].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1-b/istio-cluster-01?project=istio-project-
01
kubeconfig entry generated for istio-cluster-01.
NAME              LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
istio-cluster-01  europe-west1-b  1.10.9-gke.5    35.241.217.15  n1-standard-1  1.10.9-gke.5  4          RUNNING
```
Retrieve your credentials for kubectl:
```
~ $ gcloud container clusters get-credentials istio-cluster-01 \
  --zone europe-west1-b \
  --project istio-project-01
```
```
Fetching cluster endpoint and auth data.
kubeconfig entry generated for istio-cluster-01.
```
Grant cluster administrator (admin) permissions to the current user. To create the necessary RBAC rules for Istio, the current user requires admin permissions:
```
~ $ kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
```
```
Your active configuration is: [cloudshell-8100]
clusterrolebinding.rbac.authorization.k8s.io "cluster-admin-binding" created
```
Verify the installation of the cluster:
```
~ $ kubectl get nodes
```
```
NAME                                              STATUS    ROLES     AGE       VERSION
gke-istio-cluster-01-default-pool-4370996b-52rb   Ready     <none>    30m       v1.10.9-gke.5
gke-istio-cluster-01-default-pool-4370996b-r1j9   Ready     <none>    30m       v1.10.9-gke.5
gke-istio-cluster-01-default-pool-4370996b-rmp8   Ready     <none>    30m       v1.10.9-gke.5
gke-istio-cluster-01-default-pool-4370996b-ww4q   Ready     <none>    31m       v1.10.9-gke.5
```

## Installing the Bookinfo Application (without Istio)

Download the `bookinfo.yaml`  from the release 1.1:

```
~ $ curl -o bookinfo.yaml https://raw.githubusercontent.com/istio/istio/release-1.1/samples/bookinfo/platform/kube/bookinfo.yaml
```

```
~ $ kubectl apply -f bookinfo.yaml
```
```
service "details" created
deployment.extensions "details-v1" created
service "ratings" created
deployment.extensions "ratings-v1" created
service "reviews" created
deployment.extensions "reviews-v1" created
deployment.extensions "reviews-v2" created
deployment.extensions "reviews-v3" created
service "productpage" created
deployment.extensions "productpage-v1" created
```
Confirm all services and pods are correctly defined and running:
```
~ $ kubectl get services
```
```
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.59.247.238   <none>        9080/TCP   15s
kubernetes    ClusterIP   10.59.240.1     <none>        443/TCP    44m
productpage   ClusterIP   10.59.250.84    <none>        9080/TCP   15s
ratings       ClusterIP   10.59.243.125   <none>        9080/TCP   15s
reviews       ClusterIP   10.59.255.143   <none>        9080/TCP   15s
```
and
```
~ $ kubectl get pods
```
```
NAME                             READY     STATUS    RESTARTS   AGE
details-v1-585b6db74f-rtfqq      1/1       Running   0          3m
productpage-v1-659cfd6ff-tr5sc   1/1       Running   0          3m
ratings-v1-67dbcbddbf-l48fm      1/1       Running   0          3m
reviews-v1-6876fb6d7b-c5m4q      1/1       Running   0          3m
reviews-v2-56f9cb94b8-mlgxh      1/1       Running   0          3m
reviews-v3-78f7bbb948-jvpst      1/1       Running   0          3m
```

Then we can expose the `productpage-v1` deployment using a external load balancer service:

```
~ $ kubectl expose deployment productpage-v1 --type=LoadBalancer --port 80 --target-port 9080 --name=productpage-lb
```
```
service "productpage-lb" exposed
```
or we can expose the `productpage` service using external load balancer service:
```
~ $ kubectl expose service productpage --type=LoadBalancer --port 80 --target-port 9080 --name=productpage-lb
```
```
service "productpage-lb" exposed
```