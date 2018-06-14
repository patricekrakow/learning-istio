# Install Minishift on Linux

## Create a Linux Virtual Machine (VM) on Azure

We create an Azure VM of size *Standard D2s v3 (2 vcpus, 8 GB memory)* as it allows nested virtualization ;-)
  ```command
  user@Azure:~$ az group create --name experiment-01 --location francecentral
  user@Azure:~$ az vm create \
    --resource-group experiment-01 \
    --name minikube-01 \
    --image UbuntuLTS \
    --size Standard_D2s_v3 \
    --admin-username radicel \
    --generate-ssh-keys
  user@Azure:~$ az vm open-port --port 80 --resource-group experiment-01 --name minikube-01
  user@Azure:~$ ssh radicel@PublicIPAddress
  radicel@minikube-01:~$ exit
  ```
When you have finished, you can remove the VM and **all adjacent resources** by running:
  ```command
  user@Azure:~$ az group delete --name experiment-01
  ```
## Install VirtualBox as Hypervisor

    ```command
    $ sudo apt-get update
    $ sudo apt-get install -y virtualbox
    ```
