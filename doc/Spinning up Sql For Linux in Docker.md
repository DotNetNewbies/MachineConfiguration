# Spinning up Sql for Linux in a Docker Container

#### The Basics of Docker  

Just what is Docker and what does it do for us?  

A good question and one that is both easy to answer and difficult to understand.  

First, a bit of background as to the problem we are trying to solve.  

What you normally get:
- A single server host onto which one installs Sql Server.  
- One must guess at the required upper limit on cpu, memory, hard drive size,  
  etc.  
- Updates require shutting down the entire sql server environment, closing  
  the database, performing the update and then restarting. Access is down  
  during this maintenance period. Given that installs take a significant  
  amount of time, this is not trivial.  
- Moving the Sql environment to a new hardware platform requires a full  
  re-installation, plus migration of the corresponding databases.  

What we want:  
- A desire to have a moveable installation of Sql Server. Why? Because.  
- A desire to easily upgrade a Sql Server instance version (new one comes out,  
  I wanna upgrade!). By using containers, I can perform the update without  
  taking down the database environment by installing the new version into a new  
  image and, only when ready to go, take down the old instance and start the  
  new one. Elapsed time is a fraction of that of a standard update.  
- Ability to change the supporting hardware at a whim.  

**_NOTE: it is STRONGLY recommended that one NOT try to share volumes between_**  
**_multiple instances of Sql Server because of the way that SQL Server works_**  

SQL Server attempts to gain an exclusive lock on all its files thus trying to  
share them amongst several instances would violate those locks and generate  
errors.  

So...how does Docker accomplish this for us? First, a little prep.  

For this session, we're gonna presume that you _already have installed_ an  
instance of SSMS and have enabled WSL2 on your system. This implies that you  
are up to date on your windows updates and that WSL 2 works fine with an  
Ubuntu 20.04 (the latest LTS version as of this talk).  

Of course, you'll need to have installed Docker, duh. This talk is focused on  
the Windows users out there so we are taking the MS-desired path of installing  
our Sql Server for Linux container using the WSL2 environment so that needs to  
be installed and correctly initialized with the desired Linux environment, e.g.  
Ubuntu 20.04 (the latest LTS as of this writing).  

If you want to know just how funky you can get with your installations, the  
content and steps of this talk were all tested to execution by running under  
Windows 10 Pro on my laptop running in a VMWare virtual machine, also running  
Windows 10 Pro running Docker that is running my SQL Server for Linux instance  
under WSL 2.  

Now that the necessary support software is installed, lets continue.  

Docker is a virtualization tool *of a sort*. Docker does **_not_** virtualize  
the hardware of the host machine in the same fashion as that of a full virtual  
machine product, e.g. VMWare, does. Instead, Docker virtualizes access to the  
host operating system, allowing individual processes, e.g. SQL Server engine,  
to be executed in an _isolated_ environment.  

What does this do for us? Given that the Docker session is bound up inside a  
generated image that Docker uses to create a running instance of the process,  
we need a way of saving those image files. Said images are nothing more than  
standard operating system binary files with no special requirements as to  
placement or external format.  

Because Docker files appear as nothing more than a set of regular files to the  
host operating system, Docker instances, called "images", can be moved with a  
simple `copy` command vice some elaborate and complex, usually buggy,  
backup/restore process (yes Windows, I'm talking about you).  

There is no "install" process when starting a Docker session, one merely  
executes Docker and points Docker at the desired image to start a Docker  
session. And even better, no un-install process which needs to be executed  
should an update to the Sql Server engine be desired. One merely deletes, yes  
"deletes" the old session file, generates a new one with the new instance of  
Sql Server and you are off and running again.  

Does this require an execution of the Sql Server install process? Yes, **_ONCE_**.  
Once to create the initial image file. The even cooler thing about this is that  
single image file may be used to generate _multiple_ running instances of  
Sql Server **_with no modification, configuration, etc._**, one merely starts  
another instance of the image (see Kubernetes for details).  

Now, _generating_ that image takes a bit of work, but we'll cover that later in  
this talk.

The end result is a running instance of the desired process, executing in a  
completely isolated environment meaning that the rest of the host operating  
system _cannot_ access the running environment of the Docker session nor can  
the Docker environment "reach out and touch" the host operating system  
environment. The Docker process executes in its own "gaol" if you will.  

The particulars of Docker also lend themselves to security at a fine-grained  
level. Docker images are designed, from the outset, to be immutable. Once  
generated, the image **_CANNOT_** be changed, period. So, if you have complete  
control over the environment used to generate the initial image, you are  
guaranteed to have a clean image with no viruses, etc. installed and, once  
the image is created, **_no virus can be introduced_** under normal  
circumstances.  

"Well, what about configuration, initial set-up and stuff like that?" you might  
ask. The answer is this. Docker creates an "image" of the desired environment  
and stores said image on your hard drive, etc. Wherever it's convenient to  
store it.  

When Docker needs to create a running instance of that image, Docker will take  
that `immutable` image, wrap a _very thin_ read/write layer around the image and  
place that into memory and start it as a running process. There is no  
intelligence to this thin read/write layer, it's literally just the ability to  
read and write to the environment as desired. Could you store your database  
in this environment? Sure, however, the moment the environment is stopped,  
and the container, _not_ the image, the container, is deleted, then the  
database will disappear. The read/write layer is available at _runtime_ only.  

The result? The image remains untouched as it is used as the _source_ of the  
running instance, called a "container", but the image itself _DOES NOT RUN_.  
A _copy_ of the image is moved into memory, a thin read/write layer wrapped  
around it and this is what is placed into memory to execute. The thin layer  
on top of the running image allows for run-time configuration, etc. but the  
changes **_last only as long as the container is running_**. The moment  
the process is stopped and the container removed, those changes are  
_lost forever_ and will _not_ re-appear when the image is next wrapped into a  
new container for execution.  

So...how do we configure our SQL Server engine? It's part of generating the  
image file, thus the configuration is the same every time the container is  
executed. Need to change the configuration? **_DELETE THE IMAGE_** and create  
a new image with the desired config changes.  

**_This is the only way that persistent configuration changes can be made_**  
**_to a Docker session_**.  

"Well, that sucks!" you might be thinking. "What's the point of putting SQL  
Server into a Docker container if I'm gonna lose the info every time I shut  
the thing off and update it?". Docker has an answer for that as well:  
Docker Volumes.  

Docker volumes are chunks of the host persistence system, e.g. hard drive, that  
can be made available to a running container that _will_ persist even after a  
Docker session is terminated.  

If you know something of Linux, you might consider a Docker volume as a  
"mount point" to which one can attach some external storage device.  
Windows users will want to initially view volumes as a "network drive" in that  
one can "talk" to it but it's not part of the current system. It's attached from  
the outside.  

#### The Process  

OK, enough blather, let's build one of these beasts. What you are going to build  
is _not_ the final, end product of this talk, it's to get you going to see that  
you already have everything you need installed and it is working as desired.    

The simplest way of doing this is to use the pre-configured images as found  
on the microsoft site for such things.

After you install Docker and SSMS, then, create a volume to hold your  
database files:

`docker volume create <some volume name>`  

By default, creating a Docker volume creates a directory on the host, which is  
located in /var/lib/docker/volumes in Linux and C:\ProgramData\docker\volumes  
on Windows.  

You can inspect the result by calling:

`docker volume inspect <some volume name>`  

You'll be shown things such as the driver, the mount point, the volume name,  
etc.  

Now, understand, because Docker volumes are simply directories on the host file  
system, you can add, modify and delete files within the volume to your hearts  
content. For Linux user, remember "permissions". "root" owns the volumes so  
you'll need root access, e.g. `sudo` to access the volume (directory) content.  
You don't want to change the owner of this, Docker will become "unhappy" if you  
do.  

Wanna wipe out your volume? Execute:  

`docker volume rm <some volume name>`  

and it's gone.  

OK, we've got our volume, now lets attach it to a container and see what  
happens.  

There are several ways of attaching a volume to a container, Docker now  
recommends using the `mount` syntax even though it is more verbose, it's easier  
to understand and the order of values is not significant.  

So, a full-on docker command that leverages the standard Microsoft images along  
with the volume just created (you did create the volume, right?) is:  

`docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=P@ssw0rd" -e "MSSQL_PID=Developer" -e "MSSQL_DATA_DIR=/var/opt/mssql" -e "MSSQL_BACKUP_DIR=/var/opt/mssql/backup" -e "MSSQL_LOG_DIR=/var/opt/mssql/log" -e "MSSQL_AGENT_ENABLED=true" -p 1433:1433 --name Sql19DevLinux -d -h Sql19DevLinux --mount type=volume,source=Sql19DevLinuxV,target=/var/opt/mssql mcr.microsoft.com/mssql/server:2019-CU15-ubuntu-20.04`  

OK, let's break this monster down:

`docker run`  
- Start Docker and point it to an image to execute.  

`-e "ACCEPT_EULA=Y"`  
- Make the MS lawyers happy by accepting their EULA (End-User License Agreement)  

`-e "SA_PASSWORD=p@ssw0rd"`  
- Set the administrative password for this instance of Sql Server.  
  **_NOTE THAT THE PASSWORD MUST MEET MS SQL SERVER REQUIREMENTS_**  
  You'll get nowhere and no easy to understand error message if you don't do  
  this correctly. See the Sql Server specs for requirements on passwords.  

`-e MSSQL_PID=Developer"`  
- The container will execute the "Developer" version of Sql Server. Acceptable  
  values for this environment variable setting are:  
  - Developer
  - Express
  - Standard
  - Enterprise
  - EnterpriseCore  

`-e "MSSQL_DATA_DIR=/var/opt/mssql"`  
- The root data directory Sql Server should use to store all of its databases.  
  This value _must_ match the value provided in your "mount" command target.  
  
  **_NOTE: This one's a biggie!_**  

  If you don't set this, then Sql Server will use the default value for its  
  data storage - **_which is in the container, NOT the volume_**. Given that's  
  the case, when you next stop your container, **_ALL YOUR DATA WILL BE LOST!_**  
  so, make sure to include this setting!  

`-e "MSSQL_BACKUP_DIR=/var/opt/mssql/backup"`  
- Tell Sql Server where to store backups by default.  

`-e "MSSQL_LOG_DIR=/var/opt/mssql/log"`  
- Tell Sql Server where to store log files by default.  

`-e "MSSQL_AGENT_ENABLED=true"`  
- Enable the Sql Server Agent by default, set to `false` if you don't want  
  it activated by default.  

`-p 1433:1433`  
- Connect internal port 1433 (the default MSSQL Server port) to external port  
  1433 so that other programs can talk to the SQL Server engine.  

`--name Sql19DevLinux`  
- Pick a name that will be unique in your environment. This is the container  
  name and the name of the SQL Server engine instance.  

`-d`  
- Run the container in "detached" mode, meaning there is _no_ terminal access.  
  Folks familiar with Windows can envision this as running a "service", not  
  what's actually going on, but similar in that there is no interface to the  
  running container other than via the ports exposed during start-up.  

`-h Sql19DevLinux`  
- Container host name. The author uses the same name to save confusion.  

`mount type=volume,source=Sql19DevLinuxV,target=/var/opt/mssql`  
- type: What are we mounting  
- source: the name of the volume to attach  
- target: What is the Linux mount point  

`mcr.microsoft.com/mssql/server:2019-CU15-ubuntu-20.04`  
- What is the source image to create the container from  

OK, we get a prompt back. What does that tell us? Nothing! If you want to see  
if your container is now up and running, execute:

`docker ps`  

That command shows all running containers and yours should be listed.  

Once SSMS starts, it'll be asking for a server to connect to, use "localhost".  
Enter your user name (sa) and your super-secret-spiffy admin password and you  
should be connected!  

Note that at this time, one **_MUST_** execute SSMS and the Docker container  
_within the same host machine_, otherwise SSMS won't be able to "see" the SQL  
Server instance because the networking has not been set up for external,  
machine to machine connectivity. This process is just to get you up and running  
to make sure all the basic parts work. And this caveat applies to the situation  
where you have installed Docker on a Windows box under WSL2. Should you decide  
to do this natively on an actual Linux box running Ubuntu, one need only enter  
the ip address of the host machine to communicate with the instance running  
under Docker within the Linux environment.   

Quite a mouthful, eh? We'll break this down and include a lot of the command  
line stuff in the custom Dockerfile when we get to it so that starting one of  
these beasts will be _much_ simpler.  

So, just to see that the container volume is indeed the place that stores  
your databases, create a database e.g. "test" and add a table to it e.g.  
TestTable. Doesn't matter what content, the idea is to create a database in  
the volume and see it's not gone even when we destroy the container running  
our Linux Sql Server.  

So, now stop the container using `docker stop <the first few chars of the hash>`  
e.g. `docker stop f5e4` (Your container ID *will* be different!).  Can't  
remember your container id? Execute `docker ps`, that will tell you.  

Now that the container is stopped, remove it using `docker rm <container id>`  

Execute a `docker ps`  to see that the container is indeed done now.  

Run that massive `docker run...` command as above to re-create the container  
and start an instance of it. Note that you should get a different container id  
that the last one. This is because the container itself has been recreated  
from the static, immutable image that is referenced by the docker run command.  

Re-connect your SSMS session by dis-connecting and re-connecting to the  
database engine. Validate that your database is still there. If it isn't,  
review the commands you used to validate what you did.  

Now, is this production-ready? Oh, heck no! And yes, most will remember that I  
*really* can't stand demos that are not production-ready at some point. I make  
the exception in this case for a number of reasons:
- The Ops people are responsible for the execution environment and you don't  
  want to be stepping on bureaucratic toes. As an engineer, **_not your job!_**  
- There are many more options that can be applied to a container, some to do  
  with security, others with configuring the Sql Server Engine, etc. You'll  
  want to work with the resident DBA and your OPS people to ensure total  
  compliance with your companies' standard environment.  
- And, of course, there is the CYA aspect of it. I don't want to hear of a  
  major breach and be getting that email that says "Well, you Said... and now  
  look what's happened!" I'll repeat, I'm an *engineer*, **_NOT_** an OPS guy!  
- There are *so* many options with Docker, I want to merely introduce the  
  possibilities of what can be done with this environment, *not* impose one  
  that is the "be all and end all" for configurations. I want to just whet your  
  appetite for this, there is *so* much to explore here and this is only a  
  taste!  

So...if all has bone well, then you have successfully created a docker volume  
to store your database instances in, created, destroyed and re-created a docker  
container with the Sql Server for Linux database engine installed within and  
are now ready to see how this process might be automated and that huge command  
line can be reduced to something a bit more palatable.  

At this point, your first question aught to be: "How was that initial image  
created?". Good question, this section of the talk will walk you through the  
creation of a Dockerfile, which is nothing more than a YAML file that guides  
the Docker program creating the images.  

First, we'll create the Dockerfile from start to finish, you'll see it run and  
then we'll take it apart line by line so you know what's going on in there.  

Here is the Dockerfile:

# This Docker file will install the latest version of Sql Server 2019 into an  
# Ubuntu 20.04 desktop environment, establish a non-root user "mssql" and  
# configure Sql Server appropriately.  

# Start with a standard Ubuntu Linux install.  
# If you are feeling froggy, use Ubuntu Server or some other  # variant, however, this will work.  
FROM Ubuntu:20.04  

LABEL name="Sql Server 2019 latest"  
LABEL version="1.0.0.0"  
LABEL environment="dev/test"  
LABEL maintainer="Amaranthos Labs, LLC."  

# Now install the Ubuntu packages needed for Sql Server on
# Linux installation.  
RUN apt-get update && apt-get install -y curl apt-utils apt-transport-https software-properties-common  

# Fetch the public keys for GPG for Sql Server  
RUN curl https://packages.microsoft.com/microsoft.asc | apt-key -
# Yeah, don't forget that trailing "-" character or Bad Things will happen.

# Add the repositories for Sql Server
RUN add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/ubuntu/20/04/mssql-server-2019 focal main"
RUN apt-get update

# Now, install Sql Server
RUN apt-get install -y mssql-server  

# Setup for execution
RUN mkdir ~p /var/opt/mssql/data  
RUN chmod -R g=u /var/opt/mssql /etc/passwd  

# Create non-root user and update permissions  
RUN useradd -M -s /bin/bash -u 10001 -g 0 mssql  
RUN mkdir -p -m 770 /var/opt/mssql && chgrp -R 0 /var/opt/mssql  

# Grant sql the permissions to connect to ports <1024 as a non-root user  
RUN setcap 'cap_net_bind_service+ep' /opt/mssql/bin/sqlservr  

# Allow dumps from the non-root process  
RUN setcap 'cap_sys_ptrace+ep' /opt/mssql/bin/paldumper  
RUN setcap 'cap_sys_ptrace+ep' /usr/bin/gdb  

# Add an ldconfig file because setcap causes the os to remove LD_LIBRARY_PATH  
# and other env variables that control dynamic linking  
# The following is from the original file, the modified form is from the book.  
# RUN mkdir -p /etc/ld.so.conf.d && touch /etc/ld.so.conf.d/mssql.conf  
RUN mkdir -p -m 770 /var/ opt/ mssql && chown -R mssql: 0 /var/ opt/ mssql && chgrp -R 0 /var/ opt/ mssql  

RUN echo -e "# mssql libs\n/opt/mssql/lib" >> /etc/ld.so.conf.d/mssql.conf  
RUN ldconfig  

# Set the running user to the non-root user "mssql".  
USER mssql  

# Set the remaining environment variables  
# Modify the path to include Sql Server  
ENV PATH=${PATH}:/opt/mssql/bin  
# Setting MSSQL_PID selects which version of Sql Server to execute. The two  
# free versions are "Developer" and "Express".  
ENV MSSQL_PID=Developer  
# Not a good idea here to include the admin password. However, for the sake of
# ease, it's included here. This can be included on the command line or  
# inserted into the "secrets" portion of a Docker file which this demo does not  
# go into.  
ENV SA_PASSWORD=P@ssw0rd  

ENV ACCEPT_EULA=Y  
ENV MSSQL_DATA_DIR=/var/opt/mssql  
ENV MSSQL_LOG_DIR=/var/opt/mssql/log  
ENV MSSQL_BACKUP_DIR=/var/opt/mssql/backup  
# This works only with the "Developer" version of Sql Srver, Express doesn't  
# have an "Agent".  
ENV MSSQL_AGENT_ENABLED=true  

# Open standard Sql Server port for TCP/IP access  
EXPOSE 1433  

# Set the Volume mount point  
VOLUME /var/opt/mssql  

# Start the server  
CMD ["/opt/mssql/bin/sqlservr"]  


#### Docker command reference  

The commands presented here are _by no means_ a complete list, nor are all the  
command options presented, these commands are just those that are used within  
the confines of this talk.  

The commands are listed pretty much in order of execution, exceptions will be  
noted as appropriate.

`docker volume create <some volume name>`  
- Create a docker volume with a name of `<some volume name>` where  
  `<some volume name>` can be any appropriate name e.g. Sql19DevLinuxV.  

`docker volume inspect <some volume name>`  
- Having created the volume, inspect the details of the volume.  

`docker volume rm <some volume name>`  
- Remove a volume that is no longer needed.  

  **_ALL DATA WILL BE LOST IF THE VOLUME IS DELETED!_**  

`docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=P@ssw0rd" -e "MSSQL_PID=Developer" -e "MSSQL_DATA_DIR=/var/opt/mssql" -e "MSSQL_BACKUP_DIR=/var/opt/mssql/backup" -e "MSSQL_LOG_DIR=/var/opt/mssql/log" -e "MSSQL_AGENT_ENABLED=true" -p 1433:1433 --name Sql19DevLinux -d -h Sql19DevLinux --mount type=volume,source=Sql19DevLinuxV,target=/var/opt/mssql mcr.microsoft.com/mssql/server:2019-CU15-ubuntu-20.04`
- Start a Docker container with an instance of Sql Server 19 for Linux. The  
  command presumes the pre-existence of a volume "Sql19DevLinux" and will be  
  instantiating an instance based on the Sql Server 19 variant created for  
  Ubuntu 20.04.  

`docker ps`  
- Show all running instances of Docker containers.  

`docker stop <some container id>`  
- Stop a running container whose id is `<some container id>`. This id will be
  displayed as part of the `docker ps` command.  

`docker start <some container id>`  
- Start a docker container whose instance id is `<some container id>`. One may
  obtain the address of non-executing containers using a variant of the  
  `docker ps` command e.g. `docker ps -a`.  

#### Resources  

The source material for this talk stems from the following resources:  

<a href="https://www.amazon.com/Server-Guide-Docker-Containers-Lock/dp/1484258258">The Sql Server DBA's Guide to Docker Containers</a>

More on the guts of Sql Server on Linux:  

<a href="https://www.amazon.com/Pro-SQL-Server-Linux-Container-Based/dp/1484241274">Pro Sql Server On Linux</a>  

Sql For Linux tag list:  

<a href="https://hub.docker.com/_/microsoft-mssql-server">List of tags to use when pulling an instance of SQL Server for Linux</a>  

Raw list in JSON format:  

<a href="https://mcr.microsoft.com/v2/mssql/server/tags/list">List of tags associated with MSSQL Server for Linux</a>  

If you want to see the list outside of a browser, here is a PS command set to  
do so:  

`$repo = invoke-webrequest https://mcr.microsoft.com/v2/mssql/server/tags/list`  
`$repo.content`  

More information on the internals of Sql Server when it comes to logging and  
recovery. This is foundational information and the concepts apply to all  
commercial relational databases.   

<a href="https://docs.microsoft.com/en-us/previous-versions/technet-magazine/dd392031%2528v%253Dmsdn.10%2529">Understanding Logging and Recovery in Sql Server</a>  

Github on MSSQL Server:  

<a href="https://github.com/microsoft/mssql-docker">Github repository for MSSQL in Docker</a>  

You'll need SSMS (Sql Server Management Studio) to make your life easy, here is  
a link: <a href="https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15">SSMS</a>  

<hr />  
