# Nexus Docker Registry on CentOS 7

My goal was deployment of the the private [docker registry](https://help.sonatype.com/display/NXRM3/Private+Registry+for+Docker) on [OSS Nexus](https://www.sonatype.com/nexus-repository-oss). Since what I am describing here can be used as a blueprint for the larger scale deployments, I was doing few shortcuts, specifically: 
- I was doing everything using the KVM libvirt virtualized VMs, where hypervizor was my laptop and nothing else, hence I was limited in resources.
- I was using `/etc/hosts` entries instead of DNS resolution,
- I was generating self signed certificates, i.e. I didn't use any CA certificate,
- Entire deployment has been done behind the NAT in the libvirt KVM network `192.168.122.0/24`

I was looking for the clear `howto` on deployment of docker registry on OSS Nexus, without [Java Keytool](https://www.sslshopper.com/article-most-common-java-keytool-keystore-commands.html) and some standard things. I didn't manage to find any so I am providing this in hope that it will be useful for those who just wants to play around with OSS Nexus and it's docker registry capabilities. 

Parameters of OSS Nexus server VM.

```
VM (KVM Virtualization)
CentOS Linux release 7.4.1708 (Core) 
Memory: 6144 MB
cpu: 2 cores (vendor_id	: Intel Core Processor (Broadwell, no TSX) 4096 KB)

[root@nexus ~]# df -h
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/centos_nexus-root   44G  2.7G   42G   7% /
devtmpfs                       2.9G     0  2.9G   0% /dev
tmpfs                          2.9G     0  2.9G   0% /dev/shm
tmpfs                          2.9G  8.5M  2.9G   1% /run
tmpfs                          2.9G     0  2.9G   0% /sys/fs/cgroup
/dev/sda1                     1014M  189M  826M  19% /boot
tmpfs                          581M     0  581M   0% /run/user/0
...
```

Hostname of the server, `nexus.test.net`, IP `192.168.122.154`, network `/24`.

This is the sequence of steps I need to do for deployment of the private OSS Nexus Docker Registry:
- Deployment of OSS Nexus server [Nexus-OSS-Installation](https://github.com/hermag/Nexus-OSS-Installation);
- Creating in OSS Nexus the private, proxy and group docker repositories with corresponding ports;
- Installation of Apache web server and it's configuration as (SSL) reverse proxy for OSS Nexus private doker repository.
- Testing of deployment.

Let's get started.

## Deployment of OSS Nexus server.

Since OSS Nexus is tested and validated on Java version 8 (Java 9 is not yet recommended [PurpleBooth](https://help.sonatype.com/display/NXRM3/System+Requirements)), we need e.g. `jdk-8u151-linux-x64.rpm` from [PurpleBooth](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) as for 29.11.2017.

More precisely from here [PurpleBooth](http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.rpm) as for 29.11.2017.

Also we need the OSS Nexus itself, e.g. `nexus-3.6.1-02-unix.tar.gz` from [PurpleBooth](https://www.sonatype.com/download-oss-sonatype) as for 29.11.2017

Both of these packages suppose to be located on `/opt` partition of the server.

One also need to install git on the server and clone the [Nexus-OSS-Installation](https://github.com/hermag/Nexus-OSS-Installation) project, make `install` script executable and running it. If installation fails one can make executable the `uninstall` script and cleanup the server from nexus installation. One more thing, after installation it takes some time for nexus to start, so be patient.

## Deployment of Docker Repositories in Nexus

The default nexus admin credentials are `Username: admin`, `Password: admin123`, one need to browse to `Server Administration and Configuration` section, afterwards create blobs for repositories.
Creating blob store for private docker repository, `Type: File`, `Name: docker-private`, `Path: docker-private`. 

![GitHub Logo](/images/docker-private.png)

Creating blob store for docker repository proxy, `Type: File`, `Name: docker-proxy`, `Path: docker-proxy`. 

![GitHub Logo](/images/docker-proxy.png)

Creating blob store for docker repository group, `Type: File`, `Name: docker-group`, `Path: docker-group`. 

![GitHub Logo](/images/docker-group.png)

Eventually the list of the blob stores for docker should look like this ...

![GitHub Logo](/images/docker-blobs.png)



## Installation and Configuration of Apache Reverse Proxy

## Testing of the Setup


[Erekle Magradze](http://magradze.web.cern.ch/magradze/)

# Docker იმიჯების პირადი სანახი OSS NEXUS-ში, CentOS 7-ზე

სერვერის პარამეტრები, რომელზეც ყენდება OSS NEXUS.

```
VM (KVM Virtualization)
CentOS Linux release 7.4.1708 (Core) 
Memory: 6144 MB
cpu: 2 cores (vendor_id	: Intel Core Processor (Broadwell, no TSX) 4096 KB)

[root@nexus ~]# df -h
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/centos_nexus-root   44G  2.7G   42G   7% /
devtmpfs                       2.9G     0  2.9G   0% /dev
tmpfs                          2.9G     0  2.9G   0% /dev/shm
tmpfs                          2.9G  8.5M  2.9G   1% /run
tmpfs                          2.9G     0  2.9G   0% /sys/fs/cgroup
/dev/sda1                     1014M  189M  826M  19% /boot
tmpfs                          581M     0  581M   0% /run/user/0
...
```

## OSS Nexus სერვერის დაყენება.

რადგან OSS Nexus შემოწმებულია და დამოწმებულია Java-ს მე-8 ვერსიისთვის (Java-ს მე-9 ვერსიის გამოყენება ჯერ რეკომენდირებული არ არის იხ. [PurpleBooth](https://help.sonatype.com/display/NXRM3/System+Requirements)), საჭიროა მაგალითად შემდეგი პაკეტის `jdk-8u151-linux-x64.rpm` ჩამოტვირთვა [PurpleBooth](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) ბმულიდან 29.11.2017-ის მდგომარეობით.

უფრო კონკრეტულად კი Java-ს მე-8 ვერსიის საინსტალაციო შეიძლება ჩამოქაჩული იქნას [PurpleBooth](http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.rpm) ბმულიდან 29.11.2017-ის მდგომარეობით.

ჯავას გარდა საჭიროა თვითონ OSS Nexus, მაგალითად ერთ-ერთი ბოლო და სტაბილური ვერსია `nexus-3.6.1-02-unix.tar.gz`, რომელიც შეიძლება მოიქაჩოს შემდეგი [PurpleBooth](https://www.sonatype.com/download-oss-sonatype) ბმულიდან 29.11.2017-ის მდგომარეობით.

ჯავაც და ნექსუსის პაკეტებიც განთავსებული უნდა იყოს სერვერის `/opt` დისკზე.

## Docker იმიჯების სანახების შექმნა  OSS NEXUS-ში

## Apache Reverse Proxy ვერბ სერვერის დაყენება და გამართვა

## გამოყენების მაგალითები


[ერეკლე მაღრაძე](http://magradze.web.cern.ch/magradze/)