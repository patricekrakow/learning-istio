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
    $ apt-get install -y kubectl
    ```

## References

1. Fan, J. (2017, July 13). _Nested Virtualization in Azure._ Retrieved from https://azure.microsoft.com/en-us/blog/nested-virtualization-in-azure/
