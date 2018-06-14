# Install Minishift on Linux

## Create a Linux Virtual Machine (VM) on Azure

We create an Azure VM of size *Standard D2s v3 (2 vcpus, 8 GB memory)* as it allows nested virtualization ;-)
  ```command
  user@Azure:~$ az group create --name experiment-openshift --location francecentral
  user@Azure:~$ az vm create \
    --resource-group experiment-openshift \
    --name minishift-01 \
    --image UbuntuLTS \
    --size Standard_D2s_v3 \
    --admin-username radicel \
    --generate-ssh-keys
  user@Azure:~$ az vm open-port --port 80 --resource-group experiment-openshift --name minishift-01
  user@Azure:~$ ssh radicel@PublicIPAddress
  radicel@minikube-01:~$ exit
  ```
When you have finished, you can remove the VM and **all adjacent resources** by running:
  ```command
  user@Azure:~$ az group delete --name experiment-openshift
  ```
## Install VirtualBox as Hypervisor

  ```command
  $ sudo apt-get update
  $ sudo apt-get install -y virtualbox
  ```

## Install Minishift

  ```command
  $ wget https://github.com/minishift/minishift/releases/download/v1.12.0/minishift-1.12.0-linux-amd64.tgz
  $ tar -xvf minishift-1.12.0-linux-amd64.tgz
  $ vi bootstrap-minishift
    
  #!/bin/bash

  # add the location of minishift executable to PATH
  # I also keep other handy tools like kubectl and kubetail.sh
  # in that directory

  export MINISHIFT_HOME=~/minishift-1.12.0-linux-amd64
  export PATH=$MINISHIFT_HOME:$PATH

  minishift profile set tutorial
  minishift config set memory 8GB
  minishift config set cpus 3
  minishift config set vm-driver virtualbox
  minishift config set image-caching true
  minishift addon enable admin-user
  minishift config set openshift-version v3.7.0

  minishift start

  $ chmod +x bootstrap-minishift
  $ ./bootstrap-minishift
  $ export MINISHIFT_HOME=~/minishift-1.12.0-linux-amd64
  $ export PATH=$MINISHIFT_HOME:$PATH
  $ eval $(minishift oc-env)
  $ eval $(minishift docker-env)
  $ oc login $(minishift ip):8443 -u admin -p admin
  $ oc get node
  ```

## Install Istio

  ```command
  $ wget https://github.com/istio/istio/releases/download/0.5.1/istio-0.5.1-linux.tar.gz
  $ tar -xvf istio-0.5.1-linux.tar.gz
  $ cd istio-0.5.1
  
  $ oc adm policy add-scc-to-user anyuid -z istio-ingress-service-account \
      -n istio-system
  $ oc adm policy add-scc-to-user anyuid -z default -n istio-system
  $ oc adm policy add-scc-to-user anyuid -z prometheus -n istio-system
  ```
