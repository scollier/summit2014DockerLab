#**Lab 3: Replicate a MongoDB Database**



##**3.1 Run an Image and Look Inside**



Use Case: RHEL 6.5 base image on a RHEL 7 host.  Deploy 3 containers and set up replica set.  Configuration file, log file and database is on an external volume and authentication is possible remotely.
To make this work, Mongo needs the following repositories:
epel, ose, rhel65.  I did a reposync, brought them all local and shared them via http off the host running the containers.  I'm trying to keep everything local so it can be a disconnected environment.

This has to be done in 2 phases:

1. Set up the host with the repos, or use existing content from another host.  Create directories, etc..
2. Perform the docker steps.

Phase 1: Set up the Host
Provide access to content - RPM's.  There are two ways to handle this:

1. Synchronize repos to the localhost and share out via HTTP.

    yum install http://linux.mirrors.es.net/fedora-epel/6/i386/epel-release-6-8.noarch.rpm
    reposync --repoid=epel -l
    reposync --repoid=rhel-x86_64-server-6-ose-2.0-infrastructure -l
    

Get RHEL 6.5 from somewhere.
or
Point to repos on another host, shared out via HTTP.
2. Set up the mongodb directories:


    mkdir -p /mnt/docker/mongo/{config,mongo{1,2,3}}/{logs,db}


Create the MongoDB Docker build directory in roots home directory

    mkdir -vp /root/docker_build_dir/mongo

Contents of the Docker build directory (contents of files can be found below):

/root/docker_build_dir/mongo/  

Contents:
Dockerfile  epel.repo  mongodb.conf  ose.repo  README.md  rhel65.repo

Install dnsmasq:

    yum -y install dnsmasq

Start dnsmasq:

    systemctl start dnsmasq
    systemctl status dnsmasq

Phase 2: Build and Launch the Container.
Overall Steps:

    Build container that is not part of replica set.

    Run the containers and access remotely

    Configure the replica set if needed

Detailed Steps:

Step 1:

Copy the mongodb.conf file to the subdirectory for each container.  The mongodb.conf files must be uniq because they point to a uniq location on the filesystem, database location, and the logs.

    cp -v /root/docker_build_dir/mongo/mongodb.conf /mnt/docker/mongo/config/.

Step 2:

To build, enter the Docker build directory and run the following command:

    docker build -rm -t mongo/mongo_db .


Step 3:

Configure DNS (This _might_ be optional, the MongoDB rep is going to confirm):

MongoDB wants DNS. You must set up dnsmasq on the host running the containers, and register the IP address / hostname with it.  Otherwise, mongo can't form the replica set with IP addresses.  It needs hostnames.  There are two ways to do this:

1. Start all the containers and see what IP addresses they get, see below for more on this.
2. Disable Docker networking and set the IP address for each container manually.
This is kind of tricky to get the right IP addresses.  I launch all the containers with a hostname and name, do a docker inspect and get the IP addresses.  Then I shutdown the containers, remove them to relinquish the names, and then populate the dnsmasq info and then start them back up, in order with the right name and hostname.  Docker has been assigning the containers the same ip addresses.  Some other way might be to pull it out of docker inspect.


    echo 'address="/mongo1/172.17.0.x"' >> /etc/dnsmasq.d/0hosts
    echo 'address="/mongo2/172.17.0.x"' >> /etc/dnsmasq.d/0hosts
    echo 'address="/mongo3/172.17.0.x"' >> /etc/dnsmasq.d/0hosts


# dnsmasq configuration

    echo "listen-address=$(hostname -i)" >> /etc/dnsmasq.conf
    echo 'resolv-file=/etc/resolv.dnsmasq.conf' >> /etc/dnsmasq.conf
    echo 'conf-dir=/etc/dnsmasq.d' >> /etc/dnsmasq.conf

# External DNS server, if needed and available:

    echo 'nameserver 10.16.143.247' >> /etc/resolv.dnsmasq.conf
    
# Start the dnsmasq service

    service dnsmasq restart

Test config from any host that can access the dnsmasq host:


    dig @10.16.138.242 mongo1
    dig @10.16.138.242 mongo2
    dig @10.16.138.242 mongo3


To Test: Once the containers are running, test connectivity with telnet and get the IP address from each container.

    telnet localhost 48987   <-- to each container to verify working

    docker inspect to get IP address from each container and associate that with name mongo{1,2,3}

    docker inspect 600a6a74cfb8 | grep -i ipaddr

Step 4:
Disable Selinux for now on the host: 3/17/2014
Run each container and check the replica and replace teh --dns= with the IP address that you are running dnsmasq on:


    docker run --dns=10.16.138.242 -p 27017 -p 28018 --name=mongo1 -h mongo1 -v /mnt/docker/mongo/config:/mongo/config -v /mnt/docker/mongo1/db/:/var/lib/mongodb -v /mnt/docker/mongo1/logs:/mongo/logs -d -t mongo/mongo_db
    docker run --dns=10.16.138.242 -p 27017 -p 28018 --name=mongo2 -h mongo2 -v /mnt/docker/mongo/config:/mongo/config -v /mnt/docker/mongo2/db/:/var/lib/mongodb -v /mnt/docker/mongo2/logs:/mongo/logs -d -t mongo/mongo_db
    docker run --dns=10.16.138.242 -p 27017 -p 28018 --name=mongo3 -h mongo3 -v /mnt/docker/mongo/config:/mongo/config -v /mnt/docker/mongo3/db/:/var/lib/mongodb -v /mnt/docker/mongo3/logs:/mongo/logs -d -t mongo/mongo_db


Make sure one of the containers has a primary.
You can do this by installing the mongo package on a host. Either the host running the containers or some other host.  here are a few ways to test:
Get the port by running:


    docker ps


From a different host, give it the IP address of the server running the docker mongo container (Replace IP address with the server IP):


    mongo --host 10.16.138.199 --port 49163


From the host running the container:


    mongo --port 49203


Open a mongo client to each container.  Then you can watch for status changes when the replica set is created.  You'll see them change from > to Primary or Secondary.

Step 4.
Configure the replica set
Once you have accessed the mongo database from the mongo client, now set up the replica set.
From the mongo1 host:

    rs.initiate()
    rs.add("mongo2:27017")
    rs.add("mongo3:27017")
    rs.status()

Contents of scripts and Dockerfile:
Dockerfile:


    FROM 10.64.27.125:5000/vpavlin/rhel6
    MAINTAINER scollier <emailscottcollier@gmail.com>
    Update the system
    RUN yum -y update
    # Install the troubleshooting software
    RUN yum -y install nmap vim-enhanced net-tools telnet
    # Add repo files
    ADD ./epel.repo /etc/yum.repos.d/
    ADD ./ose.repo /etc/yum.repos.d/
    ADD ./rhel65.repo /etc/yum.repos.d/
    # Install 
    RUN yum repolist
    RUN yum -y install mongodb-server mongodb
    # Expose ports
    EXPOSE 27017 28017
    CMD ["/usr/bin/mongod", "-f", "/mongo/config/mongodb.conf"]


Repo Files:

    
    [epel]
    name=epel
    baseurl=http://10.16.138.242/epel/
    gpgcheck=0
    enabled=1
    [ose]
    name=ose
    baseurl=http://10.16.138.242/ose/
    gpgcheck=0
    enabled=1
    [rhel65]
    name=rhel65
    baseurl=http://10.16.138.242/rhel65/
    gpgcheck=0
    enabled=1


mongodb.conf:

    ##
    ### Basic Defaults
    ##
    # bind_ip = 127.0.0.1
    port = 27017
    logpath = /mongo/logs/mongodb.log
    dbpath =/var/lib/mongodb
    journal = true
    auth = false
    rest = true
    smallfiles = true
    replSet = replSet
    nohttpinterface = true


Mongo directory:

Dockerfile  
epel.repo  
mongodb.conf  
ose.repo  
README.md  
rhel65.repo

    
That's to hard, let's add a filter.

    docker inspect --format '{{ .NetworkSettings.IPAddress }}' *UUID*

    
**Lab 3 Complete!**

<!--BREAK-->

