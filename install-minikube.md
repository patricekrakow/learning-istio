# Install Minikube on Linux

## Can I Install Minikube on a Cloud Virtual Machine?

Yes, you can!

## Install kubectl

**_Warning:_** You should use a version of kubectl that is the same version (or later) as your server. Using an older kubectl with a newer server might produce validation errors.

1. ...

    ```command
    $ sudo apt-get update
    ```

2. ...

    ```command
    $ sudo apt-get install -y apt-transport-https
    ```

3. ...

    ```command
    $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    ```

4. ...

    ```command
    $ sudo bash -c 'cat >/etc/apt/sources.list.d/kubernetes.list' << EOF
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    ```

5. ...

    ```command
    $ sudo apt-get update
    ```

6. ...

    ```command
    $ sudo apt-get install -y kubectl
    ```

7. ...

    ```command
    $ echo "source <(kubectl completion bash)" >> ~/.bashrc
    ```

8. ...

    ```command
    $ kubectl version
    ```

## Install Minikube

1. Install VirtualBox as hypervisor:

    ```command
    $ sudo apt-get update
    $ sudo apt-get install -y virtualbox
    ```

2. Install Minikube

    ```command
    $ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.25.0/minikube-linux-amd64
    $ chmod +x minikube
    $ sudo mv minikube /usr/local/bin/
    ```

3. Start Minikube

    ```command
    $ minikube start
    ```

4. Check the Status of Minikube

    ```command
    $ minikube status
    ```

5. Stop Minikube

    ```command
    $ minikube stop
    ```

## Install and Configure nginx

1. Install nginx:

    ```command
    $ sudo apt-get update
    $ sudo apt-get install -y nginx
    ```

2. Configure nginx:

    ```command
    $ sudo vi /etc/nginx/sites-available/proxy
    ```
    
    and save the following configuration file:
    
    ```
    server {
      listen 80;
      location / {
        proxy_pass http://127.0.0.1:8001;
      }
    }
    ```
    
    ```command
    $ sudo rm /etc/nginx/sites-enabled/default
    $ sudo ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/proxy
    $ sudo service nginx configtest
    $ sudo service nginx reload
    
    ```

## References

1. Fan, J. (2017, July 13). _Nested Virtualization in Azure._ Retrieved from https://azure.microsoft.com/en-us/blog/nested-virtualization-in-azure/
