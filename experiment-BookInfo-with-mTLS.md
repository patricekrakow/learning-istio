# Experiment BookInfo on Istio with mTLS

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
NAME              LOCATION        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
istio-cluster-01  europe-west1-b  1.10.9-gke.5    104.155.17.101  n1-standard-1  1.10.9-gke.5  4          RUNNING
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
~ $ kubectl get node
```
```
NAME                                              STATUS    ROLES     AGE       VERSION
gke-istio-cluster-01-default-pool-c20b3c66-60fp   Ready     <none>    4m        v1.10.9-gke.5
gke-istio-cluster-01-default-pool-c20b3c66-fk2l   Ready     <none>    4m        v1.10.9-gke.5
gke-istio-cluster-01-default-pool-c20b3c66-q5ck   Ready     <none>    4m        v1.10.9-gke.5
gke-istio-cluster-01-default-pool-c20b3c66-qfnm   Ready     <none>    4m        v1.10.9-gke.5
```

## Installing Istio with mutual TLS authentication between sidecars

Download and extract the latest release fo Istio:
```
~ $ curl -L https://git.io/getLatestIstio | sh -
```
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  1456  100  1456    0     0   2261      0 --:--:-- --:--:-- --:--:--  2261
Downloading istio-1.0.5 from https://github.com/istio/istio/releases/download/1.0.5/istio-1.0.5-linux.tar.gz ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   614    0   614    0     0   2638      0 --:--:-- --:--:-- --:--:--  2635
100 14.2M  100 14.2M    0     0  7563k      0  0:00:01  0:00:01 --:--:-- 15.8M
Downloaded into istio-1.0.5:
bin  bookinfo.yaml  install  istio-1.0.5  istio.VERSION  LICENSE  README.md  samples  tools
Add /home/patrice_krakow/istio-1.0.5/bin to your path; e.g copy paste in your shell and/or ~/.profile:
export PATH="$PATH:/home/patrice_krakow/istio-1.0.5/bin"
```
Move to the Istio package directory:
```
~ $ cd istio-1.0.5
```
Add the `istioctl` client to your PATH environment variable:
```
~/istio-1.0.5 $ export PATH=$PWD/bin:$PATH
```
Check if you run `istioctl`
```
~/istio-1.0.5 $ istioctl version
```
```
Version: 1.0.5
GitRevision: c1707e45e71c75d74bf3a5dec8c7086f32f32fad
User: root@6f6ea1061f2b
Hub: docker.io/istio
GolangVersion: go1.10.4
BuildStatus: Clean
```
Install Istio’s Custom Resource Definitions via `kubectl` apply, and wait a few seconds for the CRDs to be committed in the kube-apiserver:
```
~/istio-1.0.5 $ kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
```
```
customresourcedefinition.apiextensions.k8s.io "circonuses.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "deniers.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "fluentds.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "kubernetesenvs.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "listcheckers.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "memquotas.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "noops.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "opas.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "prometheuses.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "rbacs.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "redisquotas.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "servicecontrols.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "signalfxs.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "solarwindses.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "stackdrivers.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "statsds.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "stdios.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "apikeys.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "authorizations.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "checknothings.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "kuberneteses.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "listentries.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "logentries.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "edges.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "metrics.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "quotas.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "reportnothings.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "servicecontrolreports.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "tracespans.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "rbacconfigs.rbac.istio.io" created
customresourcedefinition.apiextensions.k8s.io "serviceroles.rbac.istio.io" created
customresourcedefinition.apiextensions.k8s.io "servicerolebindings.rbac.istio.io" created
customresourcedefinition.apiextensions.k8s.io "adapters.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "instances.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "templates.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "handlers.config.istio.io" created
```
Install Istio with mutual TLS authentication between sidecars:
```
~/istio-1.0.5 $ kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```
_The following output needs to be updated, it was *not* the 1st run ;-)_
```
namespace "istio-system" configured
configmap "istio-galley-configuration" unchanged
configmap "istio-grafana-custom-resources" unchanged
configmap "istio-grafana-configuration-dashboards" unchanged
configmap "istio-grafana" unchanged
configmap "istio-statsd-prom-bridge" unchanged
configmap "prometheus" unchanged
configmap "istio-security-custom-resources" unchanged
configmap "istio" unchanged
configmap "istio-sidecar-injector" unchanged
serviceaccount "istio-galley-service-account" unchanged
serviceaccount "istio-egressgateway-service-account" unchanged
serviceaccount "istio-ingressgateway-service-account" unchanged
serviceaccount "istio-grafana-post-install-account" unchanged
clusterrole.rbac.authorization.k8s.io "istio-grafana-post-install-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-grafana-post-install-role-binding-istio-system" configured
job.batch "istio-grafana-post-install" unchanged
serviceaccount "istio-mixer-service-account" unchanged
serviceaccount "istio-pilot-service-account" unchanged
serviceaccount "prometheus" unchanged
serviceaccount "istio-cleanup-secrets-service-account" unchanged
clusterrole.rbac.authorization.k8s.io "istio-cleanup-secrets-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-cleanup-secrets-istio-system" configured
job.batch "istio-cleanup-secrets" unchanged
serviceaccount "istio-security-post-install-account" unchanged
clusterrole.rbac.authorization.k8s.io "istio-security-post-install-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-security-post-install-role-binding-istio-system" configured
job.batch "istio-security-post-install" unchanged
serviceaccount "istio-citadel-service-account" unchanged
serviceaccount "istio-sidecar-injector-service-account" unchanged
customresourcedefinition.apiextensions.k8s.io "virtualservices.networking.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "destinationrules.networking.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "serviceentries.networking.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "gateways.networking.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "envoyfilters.networking.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "httpapispecbindings.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "httpapispecs.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "quotaspecbindings.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "quotaspecs.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "rules.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "attributemanifests.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "bypasses.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "circonuses.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "deniers.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "fluentds.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "kubernetesenvs.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "listcheckers.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "memquotas.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "noops.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "opas.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "prometheuses.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "rbacs.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "redisquotas.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "servicecontrols.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "signalfxs.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "solarwindses.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "stackdrivers.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "statsds.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "stdios.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "apikeys.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "authorizations.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "checknothings.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "kuberneteses.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "listentries.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "logentries.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "edges.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "metrics.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "quotas.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "reportnothings.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "servicecontrolreports.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "tracespans.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "rbacconfigs.rbac.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "serviceroles.rbac.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "servicerolebindings.rbac.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "adapters.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "instances.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "templates.config.istio.io" configured
customresourcedefinition.apiextensions.k8s.io "handlers.config.istio.io" configured
clusterrole.rbac.authorization.k8s.io "istio-galley-istio-system" configured
clusterrole.rbac.authorization.k8s.io "istio-egressgateway-istio-system" configured
clusterrole.rbac.authorization.k8s.io "istio-ingressgateway-istio-system" configured
clusterrole.rbac.authorization.k8s.io "istio-mixer-istio-system" configured
clusterrole.rbac.authorization.k8s.io "istio-pilot-istio-system" configured
clusterrole.rbac.authorization.k8s.io "prometheus-istio-system" configured
clusterrole.rbac.authorization.k8s.io "istio-citadel-istio-system" configured
clusterrole.rbac.authorization.k8s.io "istio-sidecar-injector-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-galley-admin-role-binding-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-egressgateway-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-ingressgateway-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-mixer-admin-role-binding-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-pilot-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "prometheus-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-citadel-istio-system" configured
clusterrolebinding.rbac.authorization.k8s.io "istio-sidecar-injector-admin-role-binding-istio-system" configured
service "istio-galley" unchanged
service "istio-egressgateway" unchanged
service "istio-ingressgateway" unchanged
service "grafana" unchanged
service "istio-policy" unchanged
service "istio-telemetry" unchanged
service "istio-pilot" unchanged
service "prometheus" unchanged
service "istio-citadel" unchanged
service "servicegraph" unchanged
service "istio-sidecar-injector" unchanged
deployment.extensions "istio-galley" configured
deployment.extensions "istio-egressgateway" unchanged
deployment.extensions "istio-ingressgateway" unchanged
deployment.extensions "grafana" unchanged
deployment.extensions "istio-policy" unchanged
deployment.extensions "istio-telemetry" unchanged
deployment.extensions "istio-pilot" configured
deployment.extensions "prometheus" unchanged
deployment.extensions "istio-citadel" unchanged
deployment.extensions "servicegraph" unchanged
deployment.extensions "istio-sidecar-injector" unchanged
deployment.extensions "istio-tracing" unchanged
gateway.networking.istio.io "istio-autogenerated-k8s-ingress" configured
horizontalpodautoscaler.autoscaling "istio-egressgateway" unchanged
horizontalpodautoscaler.autoscaling "istio-ingressgateway" unchanged
horizontalpodautoscaler.autoscaling "istio-policy" unchanged
horizontalpodautoscaler.autoscaling "istio-telemetry" unchanged
horizontalpodautoscaler.autoscaling "istio-pilot" unchanged
service "jaeger-query" unchanged
service "jaeger-collector" unchanged
service "jaeger-agent" unchanged
service "zipkin" unchanged
service "tracing" unchanged
mutatingwebhookconfiguration.admissionregistration.k8s.io "istio-sidecar-injector" configured
attributemanifest.config.istio.io "istioproxy" configured
attributemanifest.config.istio.io "kubernetes" configured
stdio.config.istio.io "handler" configured
logentry.config.istio.io "accesslog" configured
logentry.config.istio.io "tcpaccesslog" configured
rule.config.istio.io "stdio" configured
rule.config.istio.io "stdiotcp" configured
metric.config.istio.io "requestcount" configured
metric.config.istio.io "requestduration" configured
metric.config.istio.io "requestsize" configured
metric.config.istio.io "responsesize" configured
metric.config.istio.io "tcpbytesent" configured
metric.config.istio.io "tcpbytereceived" configured
prometheus.config.istio.io "handler" configured
rule.config.istio.io "promhttp" configured
rule.config.istio.io "promtcp" configured
kubernetesenv.config.istio.io "handler" configured
rule.config.istio.io "kubeattrgenrulerule" configured
rule.config.istio.io "tcpkubeattrgenrulerule" configured
kubernetes.config.istio.io "attributes" configured
destinationrule.networking.istio.io "istio-policy" configured
destinationrule.networking.istio.io "istio-telemetry" configured
```
Verify the installation of Istio:
```
~/istio-1.0.5 $ kubectl get services -n istio-system
```
```
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                                                                   AGE
grafana                  ClusterIP      10.59.251.106   <none>           3000/TCP                                                                                                                  5m
istio-citadel            ClusterIP      10.59.243.246   <none>           8060/TCP,9093/TCP                                                                                                         5m
istio-egressgateway      ClusterIP      10.59.245.97    <none>           80/TCP,443/TCP                                                                                                            5m
istio-galley             ClusterIP      10.59.253.25    <none>           443/TCP,9093/TCP                                                                                                          5m
istio-ingressgateway     LoadBalancer   10.59.249.153   23.251.138.245   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:31455/TCP,8060:31068/TCP,853:30632/TCP,15030:30050/TCP,15031:31176/TCP   5m
istio-pilot              ClusterIP      10.59.240.49    <none>           15010/TCP,15011/TCP,8080/TCP,9093/TCP                                                                                     5m
istio-policy             ClusterIP      10.59.250.126   <none>           9091/TCP,15004/TCP,9093/TCP                                                                                               5m
istio-sidecar-injector   ClusterIP      10.59.250.248   <none>           443/TCP                                                                                                                   5m
istio-telemetry          ClusterIP      10.59.249.32    <none>           9091/TCP,15004/TCP,9093/TCP,42422/TCP                                                                                     5m
jaeger-agent             ClusterIP      None            <none>           5775/UDP,6831/UDP,6832/UDP                                                                                                5m
jaeger-collector         ClusterIP      10.59.251.245   <none>           14267/TCP,14268/TCP                                                                                                       5m
jaeger-query             ClusterIP      10.59.245.153   <none>           16686/TCP                                                                                                                 5m
prometheus               ClusterIP      10.59.255.107   <none>           9090/TCP                                                                                                                  5m
servicegraph             ClusterIP      10.59.242.87    <none>           8088/TCP                                                                                                                  5m
tracing                  ClusterIP      10.59.241.246   <none>           80/TCP                                                                                                                    5m
zipkin                   ClusterIP      10.59.244.28    <none>           9411/TCP                                                                                                                  5m
```
and 
```
~/istio-1.0.5 $ kubectl get pods -n istio-system
```
```
NAME                                      READY     STATUS      RESTARTS   AGE
grafana-774bf8cb47-lg6mb                  1/1       Running     0          12m
istio-citadel-cb5b884db-97nhb             1/1       Running     0          12m
istio-cleanup-secrets-m44v2               0/1       Completed   0          12m
istio-egressgateway-7bbd674db4-m2f8j      1/1       Running     0          12m
istio-galley-5b494c7f5-25mvz              1/1       Running     0          12m
istio-grafana-post-install-l4nvd          0/1       Completed   0          12m
istio-ingressgateway-5bf6c54577-6z5h5     1/1       Running     0          12m
istio-pilot-76c4dd545b-tjd2s              2/2       Running     0          12m
istio-policy-5455647857-mhfhz             2/2       Running     0          12m
istio-security-post-install-4vhxv         0/1       Completed   0          12m
istio-sidecar-injector-7f4c7db98c-tfrtl   1/1       Running     0          12m
istio-telemetry-74645487c9-rf99h          2/2       Running     0          12m
istio-tracing-ff94688bb-jpdfq             1/1       Running     0          12m
prometheus-f556886b8-9g8ns                1/1       Running     0          12m
servicegraph-b5cb7dcdd-d2bq7              1/1       Running     0          12m
```
## Installing the Bookinfo Application

Label the `default` namespace with `istio-injection=enabled`:
```
~/istio-1.0.5 $ kubectl label namespace default istio-injection=enabled
```
```
namespace "default" labeled
```
Deploy the services using `kubectl`.
But, let's first have a look at the `bookinfo.yaml` file:
```yaml
# Copyright 2017 Istio Authors
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

##################################################################################################
# Details service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: details
  labels:
    app: details
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: details
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: details-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: details
        version: v1
    spec:
      containers:
      - name: details
        image: istio/examples-bookinfo-details-v1:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
##################################################################################################
# Ratings service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: ratings
  labels:
    app: ratings
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: ratings
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ratings-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: ratings
        version: v1
    spec:
      containers:
      - name: ratings
        image: istio/examples-bookinfo-ratings-v1:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
##################################################################################################
# Reviews service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: reviews
  labels:
    app: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: reviews-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      containers:
      - name: reviews
        image: istio/examples-bookinfo-reviews-v1:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: reviews-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: reviews
        version: v2
    spec:
      containers:
      - name: reviews
        image: istio/examples-bookinfo-reviews-v2:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: reviews-v3
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: reviews
        version: v3
    spec:
      containers:
      - name: reviews
        image: istio/examples-bookinfo-reviews-v3:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
##################################################################################################
# Productpage services
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: productpage-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      containers:
      - name: productpage
        image: istio/examples-bookinfo-productpage-v1:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
```
```
~/istio-1.0.5 $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
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
~/istio-1.0.5 $ kubectl get services
```
```
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.59.253.167   <none>        9080/TCP   10s
kubernetes    ClusterIP   10.59.240.1     <none>        443/TCP    26m
productpage   ClusterIP   10.59.251.44    <none>        9080/TCP   10s
ratings       ClusterIP   10.59.252.58    <none>        9080/TCP   10s
reviews       ClusterIP   10.59.247.48    <none>        9080/TCP   10s
```
and
```
~/istio-1.0.5 $ kubectl get pods
```
```
NAME                           READY     STATUS    RESTARTS   AGE
details-v1-6865b9b99d-pv6rs    2/2       Running   0          1m
productpage-v1-f8c8fb8-2pxd6   2/2       Running   0          1m
ratings-v1-77f657f55d-zcvsq    2/2       Running   0          1m
reviews-v1-6b7f6db5c5-bfptz    2/2       Running   0          1m
reviews-v2-7ff5966b99-2sr65    2/2       Running   0          1m
reviews-v3-5df889bcff-j72rf    2/2       Running   0          1m
```
Define the ingress gateway.
But, let's first have a look at the `bookinfo-gateway.yaml` file:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```
```
~/istio-1.0.5 $ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```
```
gateway.networking.istio.io "bookinfo-gateway" created
virtualservice.networking.istio.io "bookinfo" created
```
Confirm the gateway has been created:
```
~/istio-1.0.5 $ kubectl get gateways
```
```
NAME               AGE
bookinfo-gateway   10s
```
Get the public IP address of the gateway:
```
~/istio-1.0.5 $ kubectl get svc istio-ingressgateway -n istio-system
```
```
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                                                                   AGE
istio-ingressgateway   LoadBalancer   10.59.249.153   23.251.138.245   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:31455/TCP,8060:31068/TCP,853:30632/TCP,15030:30050/TCP,15031:31176/TCP   16m
```
Copy the following URL in your browser
```
http://23.251.138.245/productpage
```
If you refresh the page several times, you should see different versions of reviews shown in productpage, presented in a round robin style (red stars, black stars, no stars), since we haven’t yet used Istio to control the version routing.

## Authorization

Enable Istio authorization for the default namespace with the `samples/bookinfo/platform/kube/rbac/rbac-config-ON.yaml` file:
```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: RbacConfig
metadata:
  name: default
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    namespaces: ["default"]
```
Apply the `samples/bookinfo/platform/kube/rbac/rbac-config-ON.yaml` file:
```
~/istio-1.0.5 $ kubectl apply -f samples/bookinfo/platform/kube/rbac/rbac-config-ON.yaml
```
```
rbacconfig.rbac.istio.io "default" created
```
By refreshing the address `http://23.251.138.245/productpage` on your browser, you should now see `RBAC: access denied`. This is because Istio authorization is “deny by default”, which means that you need to explicitly define access control policy to grant access to any service.

## Cleanup (otherwise more money will go to Google)

```
~ $ gcloud container clusters delete istio-cluster-01
```
```
The following clusters will be deleted.
 - [istio-cluster-01] in [europe-west1-b]
Do you want to continue (Y/n)?  Y
Deleting cluster istio-cluster-01...done.
Deleted [https://container.googleapis.com/v1/projects/istio-project-01/zones/europe-west1-b/clusters/istio-cluster-01].
```