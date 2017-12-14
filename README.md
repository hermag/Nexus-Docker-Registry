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

### Creation of Blobs

The default admin credentials for nexus are `Username: admin`, `Password: admin123`, one need to browse to `Server Administration and Configuration` section, afterwards create blobs for repositories.
Creating blob store for private docker repository, `Type: File`, `Name: docker-private`, `Path: docker-private`. 

![Docker blob private](/images/docker-private.png)

Creating blob store for docker repository proxy, `Type: File`, `Name: docker-proxy`, `Path: docker-proxy`. 

![Docker blob proxy](/images/docker-proxy.png)

Creating blob store for docker repository group, `Type: File`, `Name: docker-group`, `Path: docker-group`. 

![Docker blob group](/images/docker-group.png)

Eventually the list of the blob stores for docker should look like this ...

![Blobs for Docker](/images/docker-blobs.png)

### Creation of Repositories
Important note: docker daemon relies on secure communication (over SSL) to the docker registry, i.e. we need to configure Nexus docker repositories to provide the service over `HTTPS`. OSS Nexus, provides two options for having the docker registry over `HTTPS`:

1. Configure the Nexus itself using the `Java KeyTools`
2. Configure the reverse proxy web server and transfer the communication to Nexus repositories over it.

Usage of `Java KeyTools` without official `DNS` entry and without `CA` server is cumbersome and error prone, therefore I've decided to go for the second option, i.e. reverse proxy configuration of web server. Before description of apache reverse proxy configuration, I will shortly describe the options during the creation of repositories. Important point here is the usage of `http ports` instead of creation of `https connectors`, this fact simplifies overall process and it is possible because of web server reverse proxy.

Going to `Repositories` from `Blobs` in the left hand menu, I am creating the `docker-priv` repository

![Docker Priv Repo](/images/docker-priv-repository.png)

Here are the most important fields that should be filled:
- `Name: docker-priv`
- `Format: docker` (when creating the repository one should select type `docker (hosted)` as it is shown below)
![Nexus Repository Types](/images/nexus-repository-types.png)
- `Type: hosted`
- `HTTP (Create an HTTP connector at specified port. Normally used if the server is behind a secure proxy.) : 8082`
- Storage `Blob Store: docker-private`

Creating repository `docker-proxy`

Here are the most important fields that should be filled:
- `Name: docker-proxy`
- `Format: docker` (when creating the repository one should select type `docker (proxy)` as it has been shown)
- `Type: proxy`
- `HTTP (Create an HTTP connector at specified port. Normally used if the server is behind a secure proxy.) : 8083`
- Proxy `Remote Storage (Location of the remote repository being proxied): https://registry-1.docker.io`
- `Docker Index: Use Docker Hub`
- Storage `Blob Store (Blob store used to store asset contents): docker-proxy`

![Docker Proxy Repo 1](/images/docker-proxy-repo-1.png)
![Docker Proxy Repo 2](/images/docker-proxy-repo-2.png)
![Docker Proxy Repo 3](/images/docker-proxy-repo-3.png)

Creating repository `docker-group`

Here are the most important fields that should be filled:
- `Name: docker-group`
- `Format: docker` (when creating the repository one should select type `docker (group)` as it has been shown)
- `Type: group`
- `HTTP (Create an HTTP connector at specified port. Normally used if the server is behind a secure proxy.) : 8084`
- Proxy `Remote Storage (Location of the remote repository being proxied): https://registry-1.docker.io`
- Storage `Blob Store (Blob store used to store asset contents): docker-group`
- Group `Member repositories: Members` -> `docker-priv, docker-proxy`

![Docker Group Repo 1](/images/docker-group-repo-1.png)
![Docker Group Repo 2](/images/docker-group-repo-2.png)

Since, it is planned to use the following TCP ports `8081, 8082, 8083, 8084` the firewalld should be configured correspondingly, i.e.

```
firewall-cmd --permanent --add-port={8081/tcp,8082/tcp,8083/tcp,8084/tcp}

firewall-cmd --reload
```

## Installation and Configuration of Apache Reverse Proxy

Docker repositories are created in OSS Nexus, now it's time to install and configure apache reverse proxy server, which will transfer requests to `docker-priv` over `https`. Repositories, `docker-group` can be used in combination with `docker pull` and `docker push` commands, `docker-proxy` can be used for synchronization of docker images and speedup deployments.

### Installation of Apache Server

`yum install httpd mod_ssl openssl -y`

Disable `SELinux` on host and reboot it, reason for this decision is the unregistered DNS and CA and also tackle with SELinux policies configuration and making all apache related activities trusted. Since it's just a blue print, disabling of SELinux is just for shortening the deployment time. In general it is strongly NOT recommended to switch SELinux from `enforcing` mode to any other.

### Generating the self-signed certificates

On nexus server

```
cd /opt

openssl genrsa -out priv.key 2048

openssl req -new -key priv.key -out priv.csr

openssl x509 -req -days 365 -in priv.csr -signkey priv.key -out priv.crt

cp priv.crt /etc/pki/tls/certs/

cp priv.key /etc/pki/tls/private/

cp priv.csr /etc/pki/tls/private/


firewall-cmd --permanent --add-port={80/tcp,443/tcp}

firewall-cmd --permanent --add-service=http --add-service=https

firewall-cmd --reload

systemctl enable httpd

systemctl start httpd
```

For configuration, check the [ssl.conf](https://github.com/hermag/Nexus-Docker-Registry/blob/master/ssl.conf) file, particularly the  

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