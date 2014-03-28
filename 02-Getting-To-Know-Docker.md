#**Lab 1: Getting to Know Docker on RHEL**

Red Hat Enterprise Linux provides shared services for Docker. A couple of these shared services are systemd and selinux.  This lab will help to familiarize you with the common actions performed on Docker containers and images. The first part of the lab starts out on the host machine, the machine that runs all the containers.  Then we'll move on to container inspection.

##**1.1 Run an Image and Look Inside**

All actions in this lab will performed by the user *root*.

Check to ensure that SELinux is running on the host.

    getenforce

Take a look at the documetation and help that is provide by the Docker package.

    rpm -qd docker-io
    man docker
    man docker-run
    docker --help

A Docker *image* is basically a layer.  A layer never changes.  To take a look a the images that are on this system.

    docker images --help
    docker images
    
Docker provides the *run* option to run a image.  Check out the run options and then run the image.  The following command launches the image, executes the command "echo hello", and then exits.  

    docker run --help
    docker run rhel7 echo hello

You won't see any return value.  Where did it go?  Check the logs.  The following commands will list the last container that ran so you can get the UUID and check the logs.  This should return the output of "echo hello".  Finally, run with the *-t* option to allocate a psuedo-tty

    docker ps -l    
    docker logs *UUID*
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
    docker commit *Container ID* file2/container
    ae4b621fc73d0a66bf1e98657dee570043cb7f9910c0b96782a914fee85437f2
   
Now lets see if it saved the file.  Now *docker images* should show the newly commited container. Launch it again and check for the file.

    docker images
    docker run -i -t file2/container bash
    ls ~/
    exit
  
##**1.3 Run an Image and Look Around**

Now that we have explored what's on the inside of a container, let's see what is going on outside of the container.

Let's launch a container and leave it running and confirm it is running.  The *-d* option runs the container in daemon mode.  Remember, you can always get help with the options.  Run these commands on the host, you should not be inside a container at this time.

    docker run --help
    docker run -d rhel7
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
        
**Lab 1 Complete!**

<!--BREAK-->

