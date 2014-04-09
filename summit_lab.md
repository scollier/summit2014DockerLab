#**Contents**#

1. **Overview**
2. **Lab Environment**
3. **Getting to Know Docker**
4. **Containers can Talk!**

<!--BREAK-->



#**Overview of Docker on Red Hat Enterprise Linux lab**

##**1.1 Overview of Docker on Red Hat Enterprise Linux**

The rapid adoption of Docker demonstrates that the benefits of Docker and containers in general are valued by enterprise developers and administrators. Specifically, Docker and containers enable rapid application deployment by only including the minimal runtime requirements of the application. This minimal size and the mentality of replacing containers, rather than updating them, simplifies maintenance. Additionally, containers allow applications bring all of their runtime requirements with them, making them to be portable across multiple Red Hat Enterprise Linux environments. This means that containers can ease testing and troubleshooting efforts by providing a consistent runtime across development, QA and production environments. In addition, containers run applications in isolated memory, process, filesystem and networking spaces. The isolation ensures that any security breaches are limited to the container.

Red Hat has been investing in containers for a number of years in Red Hat Enterprise Linux and has been working on Docker in the upstream community since mid-2013. Red Hat's commitment to Docker and container technology is demonstrated not just in this background work, but also in the efforts to establish Docker containers as a standard part of the Red Hat Enterprise Linux environment. Red Hat has production experience leveraging container technologies like cgroups and namespaces since Red Hat Enterrise Linux 6. Establishing, consuming and sharing these capabilities as a part of Red Hat Enterprise Linux is a major step in making them consumable by enterprise customers.



##**1.2 Assumptions**

This lab manual assumes that you are attending an instructor-led training class and that you will be using this lab manual in conjunction with the lecture.

This manual also assumes that you have been granted access to a single Red Hat Enterprise Linux server with which to perform the exercises.

A working knowledge of Linux-based tools and services such as telnet, Apache, etc... are assumed.  If you do not have an understanding of any of these technologies, please let the instructors know.

##**1.3 What you can expect to learn from this training class**

Topics covered:

Starting / stopping containers<br>
Container / host exploration<br>
External Logging<br>
Saving Content <br>
Starting containers on boot<br>
Linking containers<br>
Dockerfiles



**End of Overview**

<!--BREAK-->

#**Lab 1: Getting to Know Docker on RHEL**

Red Hat Enterprise Linux provides shared services for Docker. A couple of these shared services are systemd and selinux.  This lab will help to familiarize you with the common actions performed on Docker containers and images. The first part of the lab starts out on the host machine, the machine that runs all the containers.  Then we'll move on to container inspection.

##**1.1 Run an Image and Look Inside**

All actions in this lab will performed by the user *root*.

Check to ensure that SELinux is running on the host.

    getenforce

Take a look at the documetation and general help as well as command specific help that is provide by the Docker package.

    rpm -qd docker-io
    man docker
    man docker-run
    docker --help
    docker run --help

A Docker *image* is basically a layer.  A layer never changes.  To take a look a the images that are on this system.

    docker images --help
    docker images
    
Docker provides the *run* option to run a image.  Check out the run options and then run the image.  The following command launches the image, executes the command "echo hello", and then exits.  

    docker run --help
    docker run rhel7 echo hello

You won't see any return value.  Where did it go?  Check the logs.  The following commands will list the last container that ran so you can get the UUID and check the logs.  This should return the output of "echo hello".  Finally, run with the *-t* option to allocate a psuedo-tty

    docker ps -l    
    docker logs <Container ID>
    docker run -t rhel7 echo hello

To run an interactive instance that you can look around in, pass the options *-i* and *-t*. The *-i* option starts an interactive terminal.  The *-t* option allocates a pseudo-tty. You should see different results than before.  

    docker run -i -t rhel7 bash
    
Check out the IP address of the container and also look at the route.  You can see that the default route is that of the docker0 bridge on the host.

    ip a
    ip r s
    
Grab the hostname of the container.  By default the hostname is set to the UUID of the container.  We will look at how to change that later.

    hostname
    
What processes are running inside the container?

    ps aux
        
What is the SELinux security label of the processes?

    ps -Z
    
##**1.2 Saving Content**

Now that we have an idea of what's going on inside the container, let's take a look at the process required to save a file.

Create a file inside the container and see if it persists the next time you run the container.

    echo "Hello World" >> ~/file1
    
Exit the container.

    exit
    
Run the container again and check to see if the file exists.  The file should be gone.

    docker run -i -t rhel7 bash
    ls ~/
        
Let's try this again and this time we'll commit the container.

    echo "Hello World" >> /file2
    
Exit the container and commit the container.

    exit
    docker ps -l
    docker commit <Container ID> file2/container
    ae4b621fc73d0a66bf1e98657dee570043cb7f9910c0b96782a914fee85437f2
   
Now lets see if it saved the file.  Now *docker images* should show the newly commited container. Launch it again and check for the file.

    docker images
    docker run -i -t file2/container bash
    ls ~/
    exit
  
##**1.3 Run an Image and Look Around**

Now that we have explored what's on the inside of a container, let's see what is going on outside of the container.

Let's launch a container that will run for a long time then confirm it is running.  The *-d* option runs the container in daemon mode.  Remember, you can always get help with the options.  Run these commands on the host (you should not be inside a container at this time).

    docker run --help
    docker run -d rhel7 sleep 99999
    docker ps

Check out the networking on the host. You should see the docker0 bridge and a *veth* interface attached.  The *veth* interface is one end of a virtual device that connects the container to the host machine. 

    brctl show
    
Check out the bridge and you should see that the IP address of the bridge is used as the default gateway of the container that you saw earlier.

    ip a s docker0
    
What are the firewall rules on the host?  You can see from the *nat* table that all the traffic is masqueraded so that you can reach the outside world from the containers.

    iptables -nvL
    iptables -nvL -t nat

What is Docker putting on the file system?  Check */var/lib/docker* to see what Docker actually puts down.

    ls /var/lib/docker
    
The root filesystem for the container is in the devicemapper directory.  Grab the *Container ID* and complete the path below.  Replace \<Container ID> with the output from *docker ps -l* and use tab completion to complete the \<Container ID>.

    docker ps -l
    cd /var/lib/docker/devicemapper/mnt/<Container ID><tab><tab>/rootfs
    
How do I get the IP address of a running container? Grab the \<Container ID> of a running container.

    docker ps
    docker inspect <Container ID>
    
That is quite a lot of output, let's add a filter.  Replace \<Container ID> with the output of *docker ps*.

    docker ps
    docker inspect --format '{{ .NetworkSettings.IPAddress }}' <Container ID>
    
Stop the container and check out its status. The container will not be running anymore, so it is not visible with *docker ps*.  To see the \<Container ID> of a stopped container, use the *-a* option.  The *-a* option shows all containers, started or stopped.

    docker stop <Container ID>
    docker ps
    docker ps -a
    
Start the container back up.

    docker ps -a
    docker start <Container ID>
    docker ps
    

##**1.4 Where are my logs?**

The containers do not run syslog.  In order to get logs from the container, there are a couple of methods.  The first is to run the container with */dev/log* socket bind mounted inside the container.  The other is to write to external volumes.  That's in a later lab.  Launch the container with an interactive shell.

    file /dev/log
    docker run -v /dev/log:/dev/log -i -t rhel7 bash

Now that the container is running.  Open another terminal and inspect the bind mount.

    docker ps -l
    docker inspect --format '{{.Volumes}}' <Container ID>

Go back to the original terminal. Generate a message with *logger* and exit the container.  This should write the message to the host journal.

    logger "This is a log from Summit"
    exit
    
Check the logs on the host to ensure the bind mount was successful.

    journalctl | grep -i "This is a log from Summit"
 
   
##**1.4 Control that Service!**

We can control services with systemd.  Systemd allows us to start, stop, and control which services are enabled on boot, among many other things.  In this section we will use systemd to enable the *nginx* service to start on boot.

Here is the systemd unit file that needs to be created in order for this to work.  The content below needs to be placed in the */etc/systemd/system/nginx.service* file.

    
    [Unit]
    Description=nginx server
    After=docker.service
    
    [Service]
    ExecStart=/usr/bin/docker run -d -t -p 80:80 summit/nginx
    
    [Install]
    WantedBy=multi-user.target

Now control the service.  Enable the service on reboot.

    systemctl enable nginx.service

Start the service.  When starting this service, make sure there are no other containers using port 80 or it will fail.

    docker ps
    systemctl start nginx.service
    systemctl status nginx.service

Stop the service.

    systemctl stop nginx.service
    systemctl status nginx.service

Check to ensure the service starts on boot.

    reboot
    systemctl status nginx.service
    
It's that easy!

        
**Lab 1 Complete!**

<!--BREAK-->

#**Lab 2: Containers can Talk**

Now that we have the fundamentals down, let's do something a bit more interesting with these containers.  This lab will cover launching a *MariaDB* and *Mediawiki* container.  The two will be tied together via the Docker *link* functionality.  This lab will build upon things we learned in lab 1 and expand on that.  We'll be looking at external volumes, links, and additional options to the Docker *run* command.

**A bit about links**

Straight from the Docker.io site:

"Links: service discovery for docker

Links allow containers to discover and securely communicate with each other by using the flag -link name:alias  When two containers are linked together Docker creates a parent child relationship between the containers. The parent container will be able to access information via environment variables of the child such as name, exposed ports, IP and other selected environment variables."

**Note** <br>
All images have been built before labtime.  If you would like to review what was used, all Dockerfiles are in */root/summit_link_demo*.

##**2.1 MariaDB**

This section shows how to set up an external volume and use hostnames when launching the MariaDB container.

##**2.1.1 Review the MariaDB Environment**
Review the scripts and other content that are required to build and launch the *MariaDB* container.  This lab does not require that you build the container as it has already been done to save time.  Rather, it provides the information you need to understand what the requirements of building a container like this.


    cd /root/summit_link_demo/mariadb; ls

**Review the Dockerfile**

Look at the *Dockerfile*. From the contents below, you can see that the Dockerfile is starting with the RHEL7 base image and is maintained by Stephen Tweedie.  After the *FROM* and *MAINTAINER* commands are run, the commands to install software are run with *RUN*.  Think of the *RUN* command as executing a line in a shell script.  The remaining commands are *ADD*, which are used to add content to the image and finally *EXPOSE* and *CMD* which expose ports and provide the starting command, respectively.  Exposing the port will make the port available to the *Mediawiki* container when it is launched with the *-link* command.

    # cat Dockerfile 
    FROM fedora:20
    MAINTAINER Stephen Tweedie <sct@redhat.com>
    
    RUN yum -y update; yum clean all
    RUN yum -y install mariadb-server pwgen supervisor psmisc net-tools; yum clean all
    
    VOLUME [ "/var/lib/mysql" ]
    
    ADD ./start.sh /start.sh
    ADD ./supervisord.conf /etc/supervisord.conf
    
    RUN chmod 755 /start.sh
    
    EXPOSE 3306
    
    CMD ["/bin/bash", "/start.sh"]


**Review the supervisord.conf file**

Straight from the supervisord.org site:

"Supervisor: A Process Control System

Supervisor is a client/server system that allows its users to monitor and control a number of processes on UNIX-like operating systems."

There are a couple of reasons to use *supervisord* inside a container.  The first is that Docker really only wants to be in charge of one service.  So if you are running multiple services in a POC container such as MariaDB and Apache at the same time, you need a way to manage those. Present *supervisord* as the service that runs on launch and let it control the other services in the background. Also, supervisord can run services in foreground mode.  Docker likes that.

The *supervisord.conf* file instructs the *supervisord* daemon as to which processes it is responsible for.  This *supervisord.conf* file has been pared down considerably.

    # cat supervisord.conf 
    [unix_http_server]
    file=/tmp/supervisor.sock   ; (the path to the socket file)
    
    [supervisord]
    logfile=/tmp/supervisord.log ; (main log file;default $CWD/supervisord.log)
    logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
    logfile_backups=10           ; (num of main logfile rotation backups;default 10)
    loglevel=info                ; (log level;default info; others: debug,warn,trace)
    pidfile=/tmp/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
    nodaemon=false               ; (start in foreground if true;default false)
    minfds=1024                  ; (min. avail startup file descriptors;default 1024)
    minprocs=200                 ; (min. avail process descriptors;default 200)
    
    [rpcinterface:supervisor]
    supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
    
    [supervisorctl]
    serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
    
    [program:mariadb]
    command=/usr/bin/mysqld_safe
    stdout_logfile=/var/log/supervisor/%(program_name)s.log
    stderr_logfile=/var/log/supervisor/%(program_name)s.log
    autorestart=true


**Reveiw the start.sh script**
The *start.sh* script is called by the container to start the *supervisord* daemon.  The first thing the *start.sh* script does is checks to see if the database has been created yet.  If it has, just start the container, if not, create it.  The reason for this is this container uses a shared volume.  It only needs to create the database one time.  All other times the container starts, use existing data.

    # cat start.sh 
    #!/bin/bash -x
    
    __mysql_config() {
    
    if [ ! -f /mariadb/db/ibdata1 ]; then
      echo
      echo "Database does not exist, creating now."
      echo
      sleep 2
      mysql_install_db
      chown -R mysql:mysql /var/lib/mysql
      /usr/bin/mysqld_safe & 
      sleep 10
    
      echo "Running the start_mysql function."
      mysqladmin -u root password mysqlPassword
      mysql -uroot -pmysqlPassword -e "CREATE DATABASE testdb"
      
      mysql -uroot -pmysqlPassword -e "GRANT ALL PRIVILEGES ON testdb.* \
      TO 'testdb'@'localhost' IDENTIFIED BY 'mysqlPassword'; FLUSH PRIVILEGES;"
      
      mysql -uroot -pmysqlPassword -e "GRANT ALL PRIVILEGES ON *.* \
      TO 'testdb'@'%' IDENTIFIED BY 'mysqlPassword' WITH GRANT OPTION; FLUSH PRIVILEGES;"
      
      mysql -uroot -pmysqlPassword -e "GRANT ALL PRIVILEGES ON *.* \
      TO 'root'@'%' IDENTIFIED BY 'mysqlPassword' WITH GRANT OPTION; FLUSH PRIVILEGES;"
      
      mysql -uroot -pmysqlPassword -e "select user, host FROM mysql.user;"
      killall mysqld
      sleep 10
    fi
    }
    
    __run_supervisor() {
    echo "Running the run_supervisor function."
    supervisord -n
    }
    
    # Call all functions
    __mysql_config
    __run_supervisor

    

##**2.1.1 Launch the MariaDB Container**

Either tail the audit log from your current terminal by placing the tail command in the background:

    tail -f /var/log/audit/audit.log | grep -i avc &

Or open another terminal and watch for AVCs in the foreground:


    tail -f /var/log/audit/audit.log | grep -i avc
    
Launch the container.  The directory */mariadb/db* needs to be created.  That directory will be bind mounted inside the container.

    docker run -d -v /mariadb/db:/var/lib/mysql -p 3306:3306 --name mariadb summit/mariadb


Did the container start as expected?

You will need to allow the proper SELinux permissions on the local */mariadb/db* directory so *MariaDB* can access the directory.

    ls -lZd /mariadb/db

    chcon -Rvt svirt_sandbox_file_t /mariadb/db

Now launch the container again.  First the container will have to be removed because of a naming conflict.

    docker ps -a
    docker stop <Container UID> && docker rm <Container UID>

Launch the container again.    

    docker run -d -v /mariadb/db:/var/lib/mysql -p 3306:3306 --name mariadb summit/mariadb
    docker ps -l

The container should be running at this time.



##**2.2 Mediawiki**
This section shows how to launch the *Mediawiki* container and link it back to the *MariaDB* container.

##**2.2.1 Review the Mediawiki Environment**

Review the scripts and other content that are required to build and launch the *Mediawiki* container and link it to the *MariaDB* container.  This lab does not require that you build the container as it has already been done to save time.  Rather, it provides the information you need to understand what the requirements of building a container like this.


**Reveiw the Dockerfile**


    # cat Dockerfile 
    FROM scollier/apache
    MAINTAINER Stephen Tweedie <sct@redhat.com>
    
    # Basic RPM install...
    RUN yum -y update
    
    # Install:
    #  Mediawiki, obviously
    #  php, because mediawiki doesn't by itself install php into apache
    #  php-mysqlnd: this image will be configured to run against the 
    #               Fedora-Dockerfiles mariadb image so we need the mysqld
    #               client support for php
    RUN yum -y install mediawiki php php-mysqlnd
    
    # Now wiki data.  We'll expose the wiki at $host/wiki, so the html root will be
    # at /var/www/html/wiki; to allow this to be used as a data volume we keep the
    # initialisation in a separate script.
    
    ADD config.sh /config.sh
    RUN chmod +x /config.sh
    RUN /config.sh
    
    # localhost:/wiki/mw-config should now be available to configure mediawiki.
    
    # Add script to update the IP address of a linked mariadb container if
    # needed:
    
    ADD run-mw.sh /run-mw.sh
    RUN chmod +x /run-mw.sh
    CMD ["/run-mw.sh"]








**Reveiw the config.sh script**




    # cat config.sh 
    #!/bin/bash
    #
    # The mediawiki rpm installs into /var/www/wiki.  We need to symlink this into
    # the served /var/www/html/ tree to make them visible.
    #
    # Standard config will put these in /var/www/html/wiki (ie. visible at
    # http://$HOSTNAME/wiki )
    
    mkdir -p /var/www/html/wiki
    
    cd /var/www/html/wiki
    ln -sf ../../wiki/* .
    
    # We want /var/www/html/wiki to be usable as a data volume, so it's
    # important that persistent data lives here, not in /var/www/wiki.
    
    chmod 711 .
    rm -f images
    mkdir images
    chown apache.apache images





**Reveiw the run-mw.sh script**




    # cat run-mw.sh 
    #!/bin/bash
    #
    # Run mediawiki in a docker container environment.
    
    function edit_in_place () {
        tmp=`mktemp`
        sed -e "$2" < "$1" > $tmp
        cat $tmp > "$1"
        rm $tmp
    }
    
    # If we are talking to a mariadb/mysql instance in a linked container
    # (aliased "db" on port 3306), then we need to dynamically update the
    # MW config to refer to the correct DB server IP address.
    #
    # Docker will set the DB_PORT_3306_TCP_ADDR env variable to the right
    # IP in this case.
    #
    # We'll update lines like
    #   $wgDBserver = "localhost";
    # to point to the correct location.
    
    if [ "x$DB_PORT_3306_TCP_ADDR" != "x" ] ; then
        # For initial configuration, it's also considerate to update the
        # default settings that drive the config screen defaults
        edit_in_place /usr/share/mediawiki/includes/DefaultSettings.php 's/^\$wgDBserver =.*$/\$wgDBserver = "'$DB_PORT_3306_TCP_ADDR'";/'
    
        # Only update LocalSettings if they already exist; on initial
        # setup they will not yet be here
        if [ -f /var/www/html/wiki/LocalSettings.php ] ; then
    	edit_in_place /var/www/html/wiki/LocalSettings.php 's/^\$wgDBserver =.*$/\$wgDBserver = "'$DB_PORT_3306_TCP_ADDR'";/'
        fi
    fi
    
    # Finally fall through to the apache startup script that the apache
    # Dockerfile (which we build on top of here) sets up
    exec /run-apache.sh


##**2.2.2 Launch the Mediawiki Container**

This section show's how to use hostnames and link to an existing container.  Issue the *docker run* command and link to the *mariadb* container.

**Inspect Environment variables**

Run the container in interactive mode to take a look at the environment variables.

    docker run -i -t --link mariadb:db -v /var/www/html/ -p 80:80 --name mediawiki_env summit/mediawiki bash

Once inside the container, print the envirnment variable

    env | grep DB
    
The variables should be similar to: 

    DB_NAME=/mediawiki/db
    DB_PORT=tcp://172.17.0.2:3306
    DB_PORT_3306_TCP_PORT=3306
    DB_PORT_3306_TCP_PROTO=tcp
    DB_PORT_3306_TCP_ADDR=172.17.0.2
    DB_PORT_3306_TCP=tcp://172.17.0.2:3306

This is how the linking works.  You can substitute values in your configuration files with these variables to do service discovery.  In our case, we need access to the MariaDB database and you can see that both the IP addrss of the MariaDB container and the port is listed.

Exit the container

    exit
    
Now launch the container in daemon mode for the remainder of the configuration. Notice how the options for *docker run* have changed.

    docker run -d --link mariadb:db \
    -v /var/www/html/ -p 80:80 --name mediawiki summit/mediawiki
    
Check out the link that was made.

    docker ps
    
Notice in the *NAMES* column how the link is represented.

Inspect the container and get volume information:

    docker ps

    docker inspect --format '{{ .Volumes }}' <Container ID>
    
    ls /var/lib/docker/vfs/dir/<UUID Listed from Prior Query>
    
Run the *Mediawiki* wizard and complete the configuration.

Open browser

    firefox &

Go to the *Mediawiki* home page.

    http://localhost/wiki    

Thats it.  Now you can start using your wiki.  

Now, how did this work?  The way this works is that the Dockerfile *CMD* command tells the container to launch with the *run-mw.sh* script.  Here's the key thing about what that script is doing, let's review:

    if [ "x$DB_PORT_3306_TCP_ADDR" != "x" ] ; then
        # For initial configuration, it's also considerate to update the
        # default settings that drive the config screen defaults
        edit_in_place /usr/share/mediawiki/includes/DefaultSettings.php 's/^\$wgDBserver =.*$/\$wgDBserver = "'$DB_PORT_3306_TCP_ADDR'";/'
    
        # Only update LocalSettings if they already exist; on initial
        # setup they will not yet be here
        if [ -f /var/www/html/wiki/LocalSettings.php ] ; then
    	edit_in_place /var/www/html/wiki/LocalSettings.php 's/^\$wgDBserver =.*$/\$wgDBserver = "'$DB_PORT_3306_TCP_ADDR'";/'
        fi
    fi

It's doing a check for an existing LocalSettings.php file.  We added that file during the Docker build process.  That file was copied to /var/www/html/wiki.  So, the script runs, sees that the file exists and points the *$wbDBserver* variable to the MariaDB container.  So, no matter if these containers get shut down and have new IP addresses, the Mediawiki container will always be able to find the MariaDB container because of the *link*.  That's the service discovery that's happening.

    
**Lab 2 Complete!**

<!--BREAK-->

#**Continue your Learning**

##**4.1 How to Install**

On a Fedora host

    yum install fedora-dockerfiles docker-io
    


##**4.2 More Information**

Project Atomic site:



**End of Chapter 4**

<!--BREAK-->
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

summitcontainers2014
====================

lab sessions for summit containers