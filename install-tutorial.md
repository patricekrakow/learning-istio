## Install JDK

0. Install Git:

    ```command
    $ git --version
    ```

1. Install the JDK:

    ```command
    $ sudo apt-get update
    $ sudo apt-get install -y default-jdk
    $ java -version
    $ update-alternatives --config java
    $ sudo vi /etc/environment
    
    JAVA_HOME="..."
    
    $ source /etc/environment
    $ echo $JAVA_HOME
    ```
2. Install Maven

    ```command
    $ sudo apt-get update
    $ sudo apt-get install -y maven
    $ mvn --version
    ```

3. Install Docker

    ```command
    $ sudo apt-get update
    $ sudo apt-get install \
        apt-transport-https \
        ca-certificates \
        curl \
        software-properties-common
    $ 
    ```
