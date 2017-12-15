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

If apache server was already installed, just check if the `mod_ssl` module is installed, e.g. if output of `httpd -M | grep ssl` is `ssl_module (shared)`, then module is there. 

### Generating the self-signed certificates

On nexus server

```
cd /opt

openssl genrsa -out priv.key 2048

openssl req -new -key priv.key -out priv.csr

openssl x509 -req -days 365 -in priv.csr -signkey priv.key -out priv.crt
```

Generation of key and certificate has been done, now we need to copy them in a proper place, i.e.

```
cp priv.crt /etc/pki/tls/certs/

cp priv.key /etc/pki/tls/private/

cp priv.csr /etc/pki/tls/private/
```

Now it is important to enable ports and services in `firewalld`, i.e. 

```
firewall-cmd --permanent --add-port={80/tcp,443/tcp}

firewall-cmd --permanent --add-service=http --add-service=https

firewall-cmd --reload
```

Now, for proper configuration of apache and revers proxy, check the [ssl.conf](https://github.com/hermag/Nexus-Docker-Registry/blob/master/ssl.conf) file, particularly the following sections

```
# General setup for the virtual host, inherited from global configuration
DocumentRoot "/var/www/html"

ServerName nexus.test.net:443
```

Make sure that `nexus.test.net` is accordingly changed, also make sure that `SSLCertificateFile` and `SSLCertificateKeyFile` are pointing to the existing location of the certificate and key files.

This is an example of revers proxy configuration:

```
ProxyPass / http://localhost:8082/

ProxyPassReverse / http://localhost:8082/

RequestHeader set X-Forwarded-Proto "https"
```

Which means that `https://nexus.test.net` will point to the `docker-priv` repository (check which port has been opened for `docker-priv` repository.). Once it's done, we can enable and start the apache server.

```
systemctl enable httpd

systemctl start httpd
```

If `httpd` and `nexus` daemons are running without issues, we can go to the next step, which means pushing and pulling of docker images from and to the `docker-priv` repository.

(One can use the following [link](https://wiki.centos.org/HowTos/Https) for crosschecking the `https` installation.)

## Testing of the Setup

In order to use the newly installed registry we need to trust the self signed certificate, below are the instructions for Ubuntu and for CentOS.

### Ubuntu

Creat the `nexus-docker-repo` folder for copying the certificates `sudo mkdir /usr/share/ca-certificates/test`.

Copy the self signed certificate to the newly created folder `sudo cp priv.crt /usr/share/ca-certificates/test/`.

Import and trust the certificate, `sudo dpkg-reconfigure ca-certificates`

Select `Yes` in order to agree on trusting new certificate,

![Trust cert 1](/images/add-certificate-1.png)

Scroll down and select `test/ca.crt` in my case, `ca.crt` is the name of the new certificate instead of `priv.crt`.

![Trust cert 2](/images/add-certificate-2.png)

### CentOS

Copy the `priv.crt` to `/etc/pki/ca-trust/source/anchors/` and run `update-ca-trust` it will add the new cert and update the list of trusted certificates.


### Pull and Push Docker Images

First of all make sure that docker is installed and running.

Try to log in on docker registry `docker login -u admin nexus.test.net`, it should prompt for the password, which in our case would be `admin123`.

```One can add dedicated user in OSS Nexus, which will have all rights on docker repositories. Dedicated user can be used for docker registry related manipulations.```

If log in on docker registry was successful, one can pull the image from global docker registry

```
[root@runner ~]# docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
e7bb522d92ff: Pull complete 
0f4d7753723e: Pull complete 
91470a14d63f: Pull complete 
Digest: sha256:2ffc60a51c9d658594b63ef5acfac9d92f4e1550f633a3a16d898925c4e7f5a7
Status: Downloaded newer image for nginx:latest
```

In order to push it to the registry we need to tag it accordingly

```
[root@runner ~]# docker tag nginx nexus.test.net/nginx
[root@runner ~]# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
nginx                  latest              f895b3fb9e30        3 days ago          108MB
nexus.test.net/nginx   latest              f895b3fb9e30        3 days ago          108MB
```

now we are ready to push it

```
[root@runner ~]# docker push nexus.test.net/nginx
The push refers to a repository [nexus.test.net/nginx]
995f02eaa054: Layer already exists 
938981ec0340: Layer already exists 
2ec5c0a4cb57: Layer already exists 
latest: digest: sha256:3eff18554e47c4177a09cea5d460526cbb4d3aff9fd1917d7b1372da1539694a size: 948
```

Finally it will end up in `docker-priv` repository:

As you can see there is a `nginx:latest` package in the repository,

![Docker image 1](/images/docker-image-1.png)

Here are some more details.

![Docker image 2](/images/docker-image-2.png)

## Done

[Erekle Magradze](http://magradze.web.cern.ch/magradze/)

# Docker იმიჯების პირადი სანახი OSS NEXUS-ში, CentOS 7-ზე

My goal was deployment of the the private [docker registry](https://help.sonatype.com/display/NXRM3/Private+Registry+for+Docker) on [OSS Nexus](https://www.sonatype.com/nexus-repository-oss). Since what I am describing here can be used as a blueprint for the larger scale deployments, I was doing few shortcuts, specifically:

ამ პროექტის ფარგლებში, ჩემი მიზანი იყო შემექმნა [OSS Nexus](https://www.sonatype.com/nexus-repository-oss) გამოყენებით ლოკალური/პერსონალური [docker registry](https://help.sonatype.com/display/NXRM3/Private+Registry+for+Docker). აქ აღწერილი ინსტალაციის და კონფიგურაციის ეტაპები შეიძლება გამოყენებული იქნას უფრო დიდი ინფრასტრუქტურისთვის, ამ ეტაპზე კი აქ აღწერილია მხოლოდ ძირითად ასპექტები და ზოგგან გაკეთებული მაქვს გარკვეული დაშვებები და შემოკლებები, კერძოდ:

- ინსტალაცია და კონფიგურაცია განხორციელებულია KVM libvirt ვირტუალურ მანქანაზე, სადაც ჰიპერვიზორის როლს ასრულებს ჩემი ლეპტოპი, შესაბამისად რესურსების თვალსაზრისით მაქვს გარკვეული შეზღუდვები,
- DNS ჩანაწერების ნაცვლად გამოვიყენე `/etc/hosts` ფაილი (რაც ნიქს სისტემების მცოდნეთათვის ჩვეულებრივი პრაქტიკაა),
- ინსტალაციის დროს ასევე საჭიროა სერტიფიკატები და სერტიფიკატის გასაღებები, რომელიც ასევე დავაგენერირე ლოკალურად ყოველგვარი CA გამოყენების გარეშე, 
- ნექსუსის სერვერი ისევე როგორც სხვა დანარჩენი ჰოსტები არის NAT-ის უკან რომელიც ავტომატურად იქმნება libvirt KVM ინსტალაციისას, შესაბამისად ვიყენებ სტანდარტულ `192.168.122.0/24` ქსელს.

ქვემოთ მოყვანილი ინსტრუქციის დაწერის ერთ-ერთი მთავარი მიზანი იყო რომ შემექმნა მეტ-ნაკლებად სრულყოფილი `howto` პირადი დოკერ რეგისტრის შესაქმნელად, რასაც სამწუხაროდ ვერსად მივაკვლიე. ძირითადად რეკომენდირებული იყო [Java Keytool](https://www.sslshopper.com/article-most-common-java-keytool-keystore-commands.html)-ის და სხვა სტანდარტული ხელსაწყოების გამოყენება, რაც ასევე მოითხოვს სრულფასოვანი DNS ჩანაწერისა და ავტორეზებული სერტიფიკატის/გასაღების ქონას. შეზღუდული რესურსების პირობებში ამ ყველაფრის გამართვა არც ისე იოლია, ჰოდა გადავწყვიტე გარკვეული შემოვლითი გზების გამოყენებით გამემართა ჩემი რეგისტრი და თან გამეზიარებინა ეს გამოცდილება. პირადი რეგისტრის გამოყენება ძალიან მნიშნველოვანია მუდმივად განგრძობადი კოდის განვითარებისა და მიკროსერვისების დახვეწისთვის. შეგიძლიათ ბევრი დრო აღარ დაკარგოთ ისეთ ნიუანსებზე როგორიცაა უსაფრთხოება ან ქსელის ტრაფიკი და უბრალოდ კონცენტრირდეთ ხარისხიანი კოდის წერაზე, ბოლოს კი მისცეთ პროდუქტს საბოლოო სახე.

OSS Nexus სერვერისთვის გამოყენებული ვირტუალური მანქანის პარამეტრები.

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

სერვერის სახელია `nexus.test.net`, ხოლო IP `192.168.122.154`, ქსელი `/24`.

This is the sequence of steps I need to do for deployment of the private OSS Nexus Docker Registry:
ქვემოთ მოცემულია პირადი OSS Nexus Docker Registry-ის დაყენების, გამართვის და შემოწმების ის ეტაპები რომელიც მოყვანილი იქნება ამ `howto`-ში.
- OSS Nexus server [Nexus-OSS-Installation](https://github.com/hermag/Nexus-OSS-Installation) დაყენება,
- OSS Nexus-ში docker-ის private, proxy და group  რეპოზიტორიების შექმნა;
- Apache ვებ სერვერის დაყენება და კონფიგურაცია SSL-ით სამუშაოდ და ასევე მისი, OSS Nexus კერძო doker რეპოზიტორიის რევერსულ პროქსიდ(reverse proxy) გამართვა, იგივე მიზნით შეიძლება გამოყენებული იქნას nginx-იც,
- შემოწმება და გამოყენება

ვიწყებთ, ღმერთი ჩვენსკენ :).

## OSS Nexus სერვერის დაყენება.

რადგან OSS Nexus შემოწმებულია და დამოწმებულია Java-ს მე-8 ვერსიისთვის (Java-ს მე-9 ვერსიის გამოყენება ჯერ რეკომენდირებული არ არის იხ. [PurpleBooth](https://help.sonatype.com/display/NXRM3/System+Requirements)), საჭიროა მაგალითად შემდეგი პაკეტის `jdk-8u151-linux-x64.rpm` ჩამოტვირთვა [PurpleBooth](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) ბმულიდან 29.11.2017-ის მდგომარეობით.

უფრო კონკრეტულად კი Java-ს მე-8 ვერსიის საინსტალაციო შეიძლება ჩამოქაჩული იქნას [PurpleBooth](http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.rpm) ბმულიდან 29.11.2017-ის მდგომარეობით.

ჯავას გარდა საჭიროა თვითონ OSS Nexus, მაგალითად ერთ-ერთი ბოლო და სტაბილური ვერსია `nexus-3.6.1-02-unix.tar.gz`, რომელიც შეიძლება მოიქაჩოს შემდეგი [PurpleBooth](https://www.sonatype.com/download-oss-sonatype) ბმულიდან 29.11.2017-ის მდგომარეობით.

ჯავაც და ნექსუსის პაკეტებიც განთავსებული უნდა იყოს სერვერის `/opt` დისკზე.

საჭირო ჯავა და ნექსუს პაკეტების `/opt` განაყოფზე მოთავსების შემდეგ, აუცილებელია დააყენოთ `git (yum install git -y)` და დაკლონოთ შემდეგი პროექტი [Nexus-OSS-Installation](https://github.com/hermag/Nexus-OSS-Installation), გადახვიდეთ პროექტის ფოლდერში, დაამატოთ საინსტალაციო სკრიპტს გაშვების უფლება `chmod u+x install-sonatype-nexus3.sh` და დაელოდოთ სანამ გაეშვება ნექსუსი, გაშვებას ჭირდება გარკვეული დრო (3-4 წუთი) და მოიკრიბეთ თქვენი ნებისყოფა. თუ რამე აირევა შეგიძლია ასევე დაამატოთ წაშლის სკრიპტს გაშვების უფლება `chmod u+x uninstall-sonatype-nexus3.sh` და წაშალოთ ყველაფერი რაც მანამდე იქნა დაყენებული. 

## Docker რეპოზიტორიების დაყენება OSS Nexus-ზე

### Blob-ების (სანახების) შექმნა 

ნექსუს აქვს გაჩუმების პრინციპით მინიჭებული ადმინისტრატორის სახელი და საიდუმლო სიტყვა `Username: admin`, `Password: admin123`, მას შემდეგ რაც სერვერზე გაივლით ავტორიზაციას გადადით `Server Administration and Configuration` და შეუდექით ე.წ. ბლობების შექმნას. (ბლობი წარმოადგენს რეპოსიტორიის სანახს ფაილურ სისტემაში)

პირველად ვქმნით ბლობს დოკერის კერძო რეპოსიტორიისთვის, `Type: File`, `Name: docker-private`, `Path: docker-private`. 

![Docker blob private](/images/docker-private.png)

მეორე ეტაპზე ვქმნით ბლობს დოკერის პროქსი რეპოზიტორიისთვის, `Type: File`, `Name: docker-proxy`, `Path: docker-proxy`. 

![Docker blob proxy](/images/docker-proxy.png)

საბოლოოდ ვქმნით ბლობს docker რეპოზიტორიების ჯგუფისთვის (რომელიც გვაძლევს საშუალებას რომ მივმართოთ ყველა რეპოზიტორიას ერთი URL-ით), `Type: File`, `Name: docker-group`, `Path: docker-group`. 

![Docker blob group](/images/docker-group.png)

საბოლოოდ ბლობების ჩამონათვალი უნდა გამოიყურებოდეს შემდეგნაირად ...

![Blobs for Docker](/images/docker-blobs.png)

### რეპოზიტორიების შექმნა
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

## Apache Reverse Proxy ვებ სერვერის დაყენება და კონფიგურაცია

Docker repositories are created in OSS Nexus, now it's time to install and configure apache reverse proxy server, which will transfer requests to `docker-priv` over `https`. Repositories, `docker-group` can be used in combination with `docker pull` and `docker push` commands, `docker-proxy` can be used for synchronization of docker images and speedup deployments.

### Apache Server ვებ სერვერის დაყენება

`yum install httpd mod_ssl openssl -y`

Disable `SELinux` on host and reboot it, reason for this decision is the unregistered DNS and CA and also tackle with SELinux policies configuration and making all apache related activities trusted. Since it's just a blue print, disabling of SELinux is just for shortening the deployment time. In general it is strongly NOT recommended to switch SELinux from `enforcing` mode to any other.

If apache server was already installed, just check if the `mod_ssl` module is installed, e.g. if output of `httpd -M | grep ssl` is `ssl_module (shared)`, then module is there. 

### ლოკალური (self-signed) სერტიფიკატების გენერაცია

On nexus server

```
cd /opt

openssl genrsa -out priv.key 2048

openssl req -new -key priv.key -out priv.csr

openssl x509 -req -days 365 -in priv.csr -signkey priv.key -out priv.crt
```

Generation of key and certificate has been done, now we need to copy them in a proper place, i.e.

```
cp priv.crt /etc/pki/tls/certs/

cp priv.key /etc/pki/tls/private/

cp priv.csr /etc/pki/tls/private/
```

Now it is important to enable ports and services in `firewalld`, i.e. 

```
firewall-cmd --permanent --add-port={80/tcp,443/tcp}

firewall-cmd --permanent --add-service=http --add-service=https

firewall-cmd --reload
```

Now, for proper configuration of apache and revers proxy, check the [ssl.conf](https://github.com/hermag/Nexus-Docker-Registry/blob/master/ssl.conf) file, particularly the following sections

```
# General setup for the virtual host, inherited from global configuration
DocumentRoot "/var/www/html"

ServerName nexus.test.net:443
```

Make sure that `nexus.test.net` is accordingly changed, also make sure that `SSLCertificateFile` and `SSLCertificateKeyFile` are pointing to the existing location of the certificate and key files.

This is an example of revers proxy configuration:

```
ProxyPass / http://localhost:8082/

ProxyPassReverse / http://localhost:8082/

RequestHeader set X-Forwarded-Proto "https"
```

Which means that `https://nexus.test.net` will point to the `docker-priv` repository (check which port has been opened for `docker-priv` repository.). Once it's done, we can enable and start the apache server.

```
systemctl enable httpd

systemctl start httpd
```

If `httpd` and `nexus` daemons are running without issues, we can go to the next step, which means pushing and pulling of docker images from and to the `docker-priv` repository.

(One can use the following [link](https://wiki.centos.org/HowTos/Https) for crosschecking the `https` installation.)

## გამართული სისტემის შემოწმება

In order to use the newly installed registry we need to trust the self signed certificate, below are the instructions for Ubuntu and for CentOS.

### Ubuntu

Creat the `nexus-docker-repo` folder for copying the certificates `sudo mkdir /usr/share/ca-certificates/test`.

Copy the self signed certificate to the newly created folder `sudo cp priv.crt /usr/share/ca-certificates/test/`.

Import and trust the certificate, `sudo dpkg-reconfigure ca-certificates`

Select `Yes` in order to agree on trusting new certificate,

![Trust cert 1](/images/add-certificate-1.png)

Scroll down and select `test/ca.crt` in my case, `ca.crt` is the name of the new certificate instead of `priv.crt`.

![Trust cert 2](/images/add-certificate-2.png)

### CentOS

Copy the `priv.crt` to `/etc/pki/ca-trust/source/anchors/` and run `update-ca-trust` it will add the new cert and update the list of trusted certificates.


### Docker Images მოქაჩვა და ატვირთვა

First of all make sure that docker is installed and running.

Try to log in on docker registry `docker login -u admin nexus.test.net`, it should prompt for the password, which in our case would be `admin123`.

```One can add dedicated user in OSS Nexus, which will have all rights on docker repositories. Dedicated user can be used for docker registry related manipulations.```

If log in on docker registry was successful, one can pull the image from global docker registry

```
[root@runner ~]# docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
e7bb522d92ff: Pull complete 
0f4d7753723e: Pull complete 
91470a14d63f: Pull complete 
Digest: sha256:2ffc60a51c9d658594b63ef5acfac9d92f4e1550f633a3a16d898925c4e7f5a7
Status: Downloaded newer image for nginx:latest
```

In order to push it to the registry we need to tag it accordingly

```
[root@runner ~]# docker tag nginx nexus.test.net/nginx
[root@runner ~]# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
nginx                  latest              f895b3fb9e30        3 days ago          108MB
nexus.test.net/nginx   latest              f895b3fb9e30        3 days ago          108MB
```

now we are ready to push it

```
[root@runner ~]# docker push nexus.test.net/nginx
The push refers to a repository [nexus.test.net/nginx]
995f02eaa054: Layer already exists 
938981ec0340: Layer already exists 
2ec5c0a4cb57: Layer already exists 
latest: digest: sha256:3eff18554e47c4177a09cea5d460526cbb4d3aff9fd1917d7b1372da1539694a size: 948
```

Finally it will end up in `docker-priv` repository:

As you can see there is a `nginx:latest` package in the repository,

![Docker image 1](/images/docker-image-1.png)

Here are some more details.

![Docker image 2](/images/docker-image-2.png)

## მორჩა!

[ერეკლე მაღრაძე](http://magradze.web.cern.ch/magradze/)