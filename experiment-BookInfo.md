# My Experiment with Istio on Google Could Platform using Cloud Shell

## Create a Kubernetes cluster of 4 nodes

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
WARNING: Starting in 1.12, new clusters will have basic authentication disabled by default. Basic authentication can be enabled (or disabled) manually using the `--
[no-]enable-basic-auth` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using
 the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enab
le-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a clu
ster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepa
irs.
WARNING: Starting in Kubernetes v1.10, new clusters will no longer get compute-rw and storage-ro scopes added to what is specified in --scopes (though the latter wi
ll remain included in the default --scopes). To use these scopes, add them explicitly to --scopes. To use the new behavior, set container/new_scopes_behavior proper
ty (gcloud config set container/new_scopes_behavior true).
Creating cluster istio-cluster-01 in europe-west1-b...done.
Created [https://container.googleapis.com/v1/projects/istio-project-01/zones/europe-west1-b/clusters/istio-cluster-01].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1-b/istio-cluster-01?project=istio-project-0
1
kubeconfig entry generated for istio-cluster-01.
NAME              LOCATION        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
istio-cluster-01  europe-west1-b  1.10.9-gke.5    35.241.200.101  n1-standard-1  1.10.9-gke.5  4          RUNNING
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
Your active configuration is: [cloudshell-26725]
clusterrolebinding.rbac.authorization.k8s.io "cluster-admin-binding" created
```
Verify the installation of the cluster:
```
~ $ kubectl get node
```
```
NAME                                              STATUS    ROLES     AGE       VERSION
gke-istio-cluster-01-default-pool-dc43972f-53vq   Ready     <none>    2m        v1.10.9-gke.5
gke-istio-cluster-01-default-pool-dc43972f-qz83   Ready     <none>    2m        v1.10.9-gke.5
gke-istio-cluster-01-default-pool-dc43972f-r67z   Ready     <none>    2m        v1.10.9-gke.5
gke-istio-cluster-01-default-pool-dc43972f-t76v   Ready     <none>    2m        v1.10.9-gke.5
```
## Installing Istio (without mutual TLS authentication between sidecars)

Download and extract the latest release fo Istio:
```
~ $ curl -L https://git.io/getLatestIstio | sh -
```
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  1456  100  1456    0     0   2310      0 --:--:-- --:--:-- --:--:--     0
Downloading istio-1.0.5 from https://github.com/istio/istio/releases/download/1.0.5/istio-1.0.5-linux.tar.gz ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   614    0   614    0     0   2688      0 --:--:-- --:--:-- --:--:--  2692
100 14.2M  100 14.2M    0     0  7099k      0  0:00:02  0:00:02 --:--:-- 9351k
Downloaded into istio-1.0.5:
bin  install  istio.VERSION  LICENSE  README.md  samples  tools
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
Install Istio without mutual TLS authentication between sidecars:
```
~/istio-1.0.5 $ kubectl apply -f install/kubernetes/istio-demo.yaml
```
```
namespace "istio-system" created
configmap "istio-galley-configuration" created
configmap "istio-grafana-custom-resources" created
configmap "istio-grafana-configuration-dashboards" created
configmap "istio-grafana" created
configmap "istio-statsd-prom-bridge" created
configmap "prometheus" created
configmap "istio-security-custom-resources" created
configmap "istio" created
configmap "istio-sidecar-injector" created
serviceaccount "istio-galley-service-account" created
serviceaccount "istio-egressgateway-service-account" created
serviceaccount "istio-ingressgateway-service-account" created
serviceaccount "istio-grafana-post-install-account" created
clusterrole.rbac.authorization.k8s.io "istio-grafana-post-install-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-grafana-post-install-role-binding-istio-system" created
job.batch "istio-grafana-post-install" created
serviceaccount "istio-mixer-service-account" created
serviceaccount "istio-pilot-service-account" created
serviceaccount "prometheus" created
serviceaccount "istio-cleanup-secrets-service-account" created
clusterrole.rbac.authorization.k8s.io "istio-cleanup-secrets-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-cleanup-secrets-istio-system" created
job.batch "istio-cleanup-secrets" created
serviceaccount "istio-security-post-install-account" created
clusterrole.rbac.authorization.k8s.io "istio-security-post-install-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-security-post-install-role-binding-istio-system" created
job.batch "istio-security-post-install" created
serviceaccount "istio-citadel-service-account" created
serviceaccount "istio-sidecar-injector-service-account" created
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
clusterrole.rbac.authorization.k8s.io "istio-galley-istio-system" created
clusterrole.rbac.authorization.k8s.io "istio-egressgateway-istio-system" created
clusterrole.rbac.authorization.k8s.io "istio-ingressgateway-istio-system" created
clusterrole.rbac.authorization.k8s.io "istio-mixer-istio-system" created
clusterrole.rbac.authorization.k8s.io "istio-pilot-istio-system" created
clusterrole.rbac.authorization.k8s.io "prometheus-istio-system" created
clusterrole.rbac.authorization.k8s.io "istio-citadel-istio-system" created
clusterrole.rbac.authorization.k8s.io "istio-sidecar-injector-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-galley-admin-role-binding-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-egressgateway-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-ingressgateway-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-mixer-admin-role-binding-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-pilot-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "prometheus-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-citadel-istio-system" created
clusterrolebinding.rbac.authorization.k8s.io "istio-sidecar-injector-admin-role-binding-istio-system" created
service "istio-galley" created
service "istio-egressgateway" created
service "istio-ingressgateway" created
service "grafana" created
service "istio-policy" created
service "istio-telemetry" created
service "istio-pilot" created
service "prometheus" created
service "istio-citadel" created
service "servicegraph" created
service "istio-sidecar-injector" created
deployment.extensions "istio-galley" created
deployment.extensions "istio-egressgateway" created
deployment.extensions "istio-ingressgateway" created
deployment.extensions "grafana" created
deployment.extensions "istio-policy" created
deployment.extensions "istio-telemetry" created
deployment.extensions "istio-pilot" created
deployment.extensions "prometheus" created
deployment.extensions "istio-citadel" created
deployment.extensions "servicegraph" created
deployment.extensions "istio-sidecar-injector" created
deployment.extensions "istio-tracing" created
gateway.networking.istio.io "istio-autogenerated-k8s-ingress" created
horizontalpodautoscaler.autoscaling "istio-egressgateway" created
horizontalpodautoscaler.autoscaling "istio-ingressgateway" created
horizontalpodautoscaler.autoscaling "istio-policy" created
horizontalpodautoscaler.autoscaling "istio-telemetry" created
horizontalpodautoscaler.autoscaling "istio-pilot" created
service "jaeger-query" created
service "jaeger-collector" created
service "jaeger-agent" created
service "zipkin" created
service "tracing" created
mutatingwebhookconfiguration.admissionregistration.k8s.io "istio-sidecar-injector" created
attributemanifest.config.istio.io "istioproxy" created
attributemanifest.config.istio.io "kubernetes" created
stdio.config.istio.io "handler" created
logentry.config.istio.io "accesslog" created
logentry.config.istio.io "tcpaccesslog" created
rule.config.istio.io "stdio" created
rule.config.istio.io "stdiotcp" created
metric.config.istio.io "requestcount" created
metric.config.istio.io "requestduration" created
metric.config.istio.io "requestsize" created
metric.config.istio.io "responsesize" created
metric.config.istio.io "tcpbytesent" created
metric.config.istio.io "tcpbytereceived" created
prometheus.config.istio.io "handler" created
rule.config.istio.io "promhttp" created
rule.config.istio.io "promtcp" created
kubernetesenv.config.istio.io "handler" created
rule.config.istio.io "kubeattrgenrulerule" created
rule.config.istio.io "tcpkubeattrgenrulerule" created
kubernetes.config.istio.io "attributes" created
destinationrule.networking.istio.io "istio-policy" created
destinationrule.networking.istio.io "istio-telemetry" created
```
Verify the installation of Istio:
```
~/istio-1.0.5 $ kubectl get services -n istio-system
```
```
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                                                                                   AGE
grafana                  ClusterIP      10.59.244.102   <none>         3000/TCP                                                                                                                  9m
istio-citadel            ClusterIP      10.59.249.109   <none>         8060/TCP,9093/TCP                                                                                                         9m
istio-egressgateway      ClusterIP      10.59.243.234   <none>         80/TCP,443/TCP                                                                                                            9m
istio-galley             ClusterIP      10.59.246.22    <none>         443/TCP,9093/TCP                                                                                                          9m
istio-ingressgateway     LoadBalancer   10.59.253.22    35.195.51.41   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:31318/TCP,8060:30907/TCP,853:31558/TCP,15030:30849/TCP,15031:31876/TCP   9m
istio-pilot              ClusterIP      10.59.243.51    <none>         15010/TCP,15011/TCP,8080/TCP,9093/TCP                                                                                     9m
istio-policy             ClusterIP      10.59.248.39    <none>         9091/TCP,15004/TCP,9093/TCP                                                                                               9m
istio-sidecar-injector   ClusterIP      10.59.241.62    <none>         443/TCP                                                                                                                   9m
istio-telemetry          ClusterIP      10.59.250.205   <none>         9091/TCP,15004/TCP,9093/TCP,42422/TCP                                                                                     9m
jaeger-agent             ClusterIP      None            <none>         5775/UDP,6831/UDP,6832/UDP                                                                                                9m
jaeger-collector         ClusterIP      10.59.254.40    <none>         14267/TCP,14268/TCP                                                                                                       9m
jaeger-query             ClusterIP      10.59.246.199   <none>         16686/TCP                                                                                                                 9m
prometheus               ClusterIP      10.59.254.88    <none>         9090/TCP                                                                                                                  9m
servicegraph             ClusterIP      10.59.247.209   <none>         8088/TCP                                                                                                                  9m
tracing                  ClusterIP      10.59.248.76    <none>         80/TCP                                                                                                                    9m
zipkin                   ClusterIP      10.59.252.106   <none>         9411/TCP                                                                                                                  9m
```
and 
```
~/istio-1.0.5 $ kubectl get pods -n istio-system
```
```
NAME                                      READY     STATUS      RESTARTS   AGE
grafana-774bf8cb47-pspcs                  0/1       ContainerCreating   0          35s
istio-citadel-cb5b884db-pqb6m             1/1       Running             0          34s
istio-cleanup-secrets-744hx               1/1       Running             0          37s
istio-egressgateway-dc49b5b47-4blmr       0/1       ContainerCreating   0          35s
istio-galley-5b494c7f5-2rg2j              0/1       ContainerCreating   0          35s
istio-grafana-post-install-cbtfx          1/1       Running             0          37s
istio-ingressgateway-64cb7d5f6d-8sczc     0/1       ContainerCreating   0          35s
istio-pilot-549755bccb-rklf8              0/2       ContainerCreating   0          34s
istio-policy-858884d9c-v77cg              0/2       ContainerCreating   0          35s
istio-security-post-install-z8sdh         1/1       Running             0          37s
istio-sidecar-injector-7f4c7db98c-q2tqm   0/1       ContainerCreating   0          33s
istio-telemetry-748d58f6c5-4l6dh          0/2       ContainerCreating   0          34s
istio-tracing-ff94688bb-pjphg             1/1       Running             0          33s
prometheus-f556886b8-w4g5f                1/1       Running             0          34s
servicegraph-b5cb7dcdd-2zb24              1/1       Running             0          33s
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
details       ClusterIP   10.59.250.34    <none>        9080/TCP   4m
kubernetes    ClusterIP   10.59.240.1     <none>        443/TCP    24m
productpage   ClusterIP   10.59.244.150   <none>        9080/TCP   4m
ratings       ClusterIP   10.59.245.117   <none>        9080/TCP   4m
reviews       ClusterIP   10.59.240.215   <none>        9080/TCP   4m
```
and
```
~/istio-1.0.5 $ kubectl get pods
```
```
NAME                           READY     STATUS    RESTARTS   AGE
details-v1-6865b9b99d-lvht8    2/2       Running   0          5m
productpage-v1-f8c8fb8-llfgn   2/2       Running   0          5m
ratings-v1-77f657f55d-7kcst    2/2       Running   0          5m
reviews-v1-6b7f6db5c5-58q65    2/2       Running   0          5m
reviews-v2-7ff5966b99-cn297    2/2       Running   0          5m
reviews-v3-5df889bcff-cvtxg    2/2       Running   0          5m
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
bookinfo-gateway   1m
```
Get the public IP address of the gateway:
```
~/istio-1.0.5 $ kubectl get svc istio-ingressgateway -n istio-system
```
```
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                                                                                      
                             AGE
istio-ingressgateway   LoadBalancer   10.59.243.24   23.251.138.245   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:31570/TCP,8060:31579/TCP,853:32744/TCP,15030
:31970/TCP,15031:31202/TCP   29m
```
Copy the following URL in your browser
```
http://23.251.138.245/productpage
```
If you refresh the page several times, you should see different versions of reviews shown in productpage, presented in a round robin style (red stars, black stars, no stars), since we haven’t yet used Istio to control the version routing.

## Configuring the Bookinfo Application

### Routing Rules

Create default destination rules for the Bookinfo services.
But, let's first have a look at the `destination-rule-all.yaml` file:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v2-mysql
    labels:
      version: v2-mysql
  - name: v2-mysql-vm
    labels:
      version: v2-mysql-vm
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
```
```
~/istio-1.0.5 $ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```
```
destinationrule.networking.istio.io "productpage" created
destinationrule.networking.istio.io "reviews" created
destinationrule.networking.istio.io "ratings" created
destinationrule.networking.istio.io "details" created
```