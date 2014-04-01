#**Lab 2: Containers can Talk**

Now that we have the fundamentals down, let's do something a bit more interesting with these containers.  This lab will cover launching a *MariaDB* and *Mediawiki* container.  The two will be tied together via the Docker *link* functionality.  This lab will build upon things we learned in lab 1 and expand on that.  We'll be looking at external volumes, links, and additional options to the Docker *run* command.


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

    docker run -d -v /mariadb/db:/var/lib/mysql -p 3306:3306 --name mariadb summit/mariadb


Did the container start as expected?

You will need to allow the proper SELinux permissions on the local */mariadb/db* directory so *MariaDB* can access the directory.

    ls -lZd /mariadb/db

    chcon -Rvt svirt_sandbox_file_t /mariadb/db

Now launch the container again.  First the container will have to be removed because of a naming conflict.

    docker ps -a
    docker stop <Container UID> && docker rm <Container UID>
    docker run -d -v /mariadb/db:/var/lib/mysql -p 3306:3306 --name mariadb summit/mariadb
    docker ps -l

The container should be running at this time.



##**2.2 Mediawiki**
This section shows how to launch the *Mediawiki* container and link back to the *MariaDB* container.

##**2.2.1 Launch the Mediawiki Container**

This section show's how to use hostnames and link to an existing container.

    docker run -d --link mariadb:db \
    -v /var/www/html/ -p 80:80 -name mediawiki summit/mediawiki
    
Inspect the container and get volume information:

    docker inspect --format '{{ .Volumes }}' <Container ID>
    
    ls /var/lib/docker/vfs/dir/<UUID Listed from Prior Query>
    
Run the *Mediawiki* configuration

1. Open browser
2. http://localhost/wiki
3. wizard...
4. ...
5. Once finished, copy the *LocalSettings.php* file that was downloaded to the bind mounted volume.

    cp LocalSettings.php /var/lib/docker/vfs/dir/<UUID Listed from Prior Query>
    
6. Complete the install by entering the wiki.


    
**Lab 2 Complete!**

<!--BREAK-->

