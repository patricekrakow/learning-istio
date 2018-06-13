## Install JDK

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
