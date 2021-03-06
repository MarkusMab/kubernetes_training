# Building Your Work Environment 
## Install Docker
Note: The commands in the guide will be Debian based (Ubuntu).  However, Docker is not limited to just this distribution and can be downloaded on most other distributions, as well as MAC OS, and Windows 10 platform. 

1. First we will update our package list and install docker community edition.
```
sudo apt-get update
sudo apt-get install docker-ce 
```

2. Start the Docker service
```
Sudo service docker start 
```

3. Start the first container.  There is a simple hello world container on Docker hub that we can start and test.

```
docker run hello-world
```

Output should be:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete
Digest: sha256:97ce6fa4b6cdc0790cda65fe7290b74cfebd9fa0c9b8c38e979330d547d22ce1
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```
 
 
## Creating our first Docker container

We have used a default container from Docker hub, but lets make our own container.  We will use Centos for our package source and start from there.  We will then install a number of packages inside of it and then expose port 80 and 443.  Finally we will specify the executable to start when the container comes up.  

1. Clone out the containers of the the class files from github and we will start in the web area.

```
git clone https://github.com/sealingtech/kubernetes_training
cd kubernetes_training/docker/web/
```


2. There is a file called Dockerfile with the following contents (note capital "D" is important).  Look at the contents to get an understanding of the contents.  Run the command:

```
cat Dockerfile
```

Output:

```
#Start with Centos for the package source
FROM centos:centos7
#Who maintains the image (could be your email)
MAINTAINER Stech Training
#Best practice to always update
RUN yum -y update
#Install Apache
RUN yum -y install httpd
#Install PHP and some addons
RUN yum -y install php \
    php-pdo \
    php-gd \
    php-mysqli \
    php-mbstring
#Delete what is currently there
RUN rm -rf /var/www/html/*
#Copy the contents inside your current directory on your local build into the container
COPY html/ /var/www/html/
#This will fix up the Apache configuration to ensure logs to to STDOUT
COPY httpd.conf /etc/httpd/conf/httpd.conf
#We will expose two networking ports to the container
EXPOSE 80 443
#When the container starts start up Apache as the process
CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND"]
```

2. Build the container.  This will first download the Centos image (if it hasn't been downloaded already) and then begin executing then begin executing the commands in your Dockerfile.  When this is done it will save off an image into your local Docker repository called stech/apache.  We will then view our image repository.

```
sudo docker build -t stech/apache . 
sudo docker image ls
```

3. Start up the container.  The arguments are as follows
  + -t Give the container a tty (a terminal) so you can execute a shell
  + -d Detached, run in the background
  + --name give the container the name apache
  + -p Map port 80 on the local host port to the container port 80.  Your local host will listen on port 80 and forward all requests to your Docker container also listening on port 80.  When we move the Docker container to Kubernetes we will use  more robust networking options, but this works for testing.

```
#Create a user defined bridged network
docker network create stech
#Create our container
sudo docker run -td --name apache --net stech -p 80:80 stech/apache 
# show your container running
sudo docker ps
```

4. The containers is now running in the background, you can access the container's terminal

```
sudo docker exec -it apache bash  
```

5. To escape from the terminal you press ctrl+a, then while still holding ctrl, hit d

6. Open your web browser and browse to http://127.0.0.1/ and you should see apache serving from your container.

## Building MariaDB container.
For the database, we will simply pull from Docker hub already made images instead of building our own container.  The benefit to doing this is they are maintained inside and Docker hub and they generally have options made available to set.  The options are set as environment variables and then scripts are used to configure the container accordingly.  You can see the options available here:
https://hub.docker.com/_/mariadb/

1.  cd over to our db folder in the files we downloaded
```
cd ../db
```

2. View the contents of our Dockerfile
```
cat Dockerfile
```
Output:

```
FROM mariadb:10.3

#Set the environment variables the container uses
ENV MYSQL_ROOT_PASSWORD password12345
ENV MYSQL_DATABASE applications
ENV MYSQL_USER docker_man
ENV MYSQL_PASSWORD docker12345

#Copy SQL commands to create the tables needed for out web app
COPY createtable.sql /docker-entrypoint-initdb.d/
```

3. Build the container:
```
docker build -t stech/mariadb .
```

4. Run the following commands to create a container called mariadb using the mariadb image from Docker Hub.
```
docker run -itd --name mariadb --net stech stech/mariadb
```

5. Lets get a shell to see how the container was configured
```
docker exec -it mariadb bash
```

6. Inside the container we can see how environment variables were set from the options we set on the command line:
```
env
```
Will give us the output:

```
MARIADB_MAJOR=10.2
HOSTNAME=a5f02b5d5819
TERM=xterm
MYSQL_DATABASE=applications
MYSQL_PASSWORD=docker12345
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
GPG_KEYS=199369E5404BD5FC7D2FE43BCBCB082A1BB943DB 	430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A 	4D1BB29D63D98E422B2113B19334A25F8507EFA5
PWD=/
SHLVL=1
HOME=/root
GOSU_VERSION=1.10
MYSQL_USER=docker_man
MARIADB_VERSION=10.2.13+maria~jessie
MYSQL_ROOT_PASSWORD=password12345
_=/usr/bin/env
```

7. View processes, notice that the container has very little going on, when we aren't accessing the shell only Mysqld would be running:
```
ps -ef
```
output:

```
UID        PID  PPID  C STIME TTY          TIME CMD
mysql        1     0  0 17:11 pts/0    00:00:00 mysqld
root       177     0  0 17:13 pts/1    00:00:00 bash
root       182   177  0 17:16 pts/1    00:00:00 ps -ef
```

8. To escape from the terminal you press ctrl+a, then while still holding ctrl, hit d

## Connecting the two containers
We haven't really looked at networking up until this point.  We have been connecting the docker containers to a bridge we created called stech.  All containers on this network can communicate to one another as if they were all connected to a switch.  Notice that we haven't set IPs, netmasks or any of the details up until this point. The way we can lookup container IPs is through a DNS server that Docker runs.  We are able to lookup the names of containers through DNS to get there IPs.  The networking will change when we send these containers to the cloud using Kubernetes, but this works well for testing.

1. To view docker networking information

```
docker network inspect stech
```

2. Lets pull up the apache containers terminal and run ping to mariadb to make sure it is working (ctrl-c when done)

```
docker exec -it apache bash
ping mariadb
```

3. You can see that our simple web application is configured to lookup the mariadb hostname.  

```
cat /var/www/html/config/config.php
```

4.  Lets make sure our simple web application works open up a web browser and go to (remember that we mapped localhost port 80 to port 80 inside of the container when we ran the run command):
http://127.0.0.1/applications.html

5. Enter in information in the web app and then select "submit".    You should get the message "Thanks for your application!" which verifies we have written data to the database.