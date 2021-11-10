---
layout: post
title:  "Total Recall Docker 102"
date:   2020-05-04
categories: [blog]
tags: [docker]
excerpt_separator: <!--more-->
---

My jog down the Docker lane continues, in my earlier post [total-recall-docker-101]({% post_url 2020-05-03-total-recall-docker-101 %}), I skimmed through basic Docker commands.
Those were ad hoc commands, simply run from shell and I had containers running. I am going to try Dockerfile now to create an image of my own with Ansible installed.
Once I have the image, I will then spin up containers.
<!--more-->

{% highlight docker %}

FROM centos:centos7.7.1908
RUN yum check-update; \
yum install -y gcc libffi-devel python-devel openssl-devel epel-release; \
yum install -y python-pip python-wheel; \
yum install -y openssh-clients; \
yum install -y ansible

{% endhighlight %}

{% highlight bash %}

$ docker build --rm -t centos:ansible .                                                                                                                                                       
Sending build context to Docker daemon  997.4kB
Step 1/2 : FROM centos:centos7.7.1908
centos7.7.1908: Pulling from library/centos
f34b00c7da20: Pull complete 
Digest: sha256:50752af5182c6cd5518e3e91d48f7ff0cba93d5d760a67ac140e2d63c4dd9efc
Status: Downloaded newer image for centos:centos7.7.1908
 ---> 08d05d1d5859
Step 2/2 : RUN yum check-update; yum install -y gcc libffi-devel python-devel openssl-devel epel-release; yum install -y python-pip python-wheel; yum install -y openssh-clients; yum install -y ansible
 ---> Running in c60e839dffc9
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: d36uatko69830t.cloudfront.net
 * extras: d36uatko69830t.cloudfront.net
 * updates: d36uatko69830t.cloudfront.net

<truncated>

Complete!
Removing intermediate container c60e839dffc9
 ---> 70c1d52ddda5
Successfully built 70c1d52ddda5
Successfully tagged centos:ansible

$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              ansible             70c1d52ddda5        18 seconds ago      604MB
lambci/lambda       python3.6           9831f2ce9220        2 weeks ago         882MB
lambci/lambda       python2.7           3e74a4f229dc        2 weeks ago         752MB
lambci/lambda       python3.8           4de52bf902df        3 weeks ago         516MB
lambci/lambda       python3.7           173094877052        3 weeks ago         933MB
lambci/lambda       nodejs8.10          27ca855c45b0        3 weeks ago         811MB
lambci/lambda       nodejs12.x          6cafc2b52f33        4 weeks ago         380MB
lambci/lambda       nodejs10.x          729a6cecec71        4 weeks ago         376MB
centos              latest              470671670cac        3 months ago        237MB
centos              centos7.7.1908      08d05d1d5859        5 months ago        204MB

$ docker run --rm -t -d  --name ansible01 centos:ansible
a2a03b17bc44a3f082944c190ca11bf6dcf82376b042c44acfddebdf85bb5e12

$ docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a2a03b17bc44        centos:ansible      "/bin/bash"         4 seconds ago       Up 3 seconds                            ansible01

$ docker exec -it ansible01 /bin/bash
[root@a2a03b17bc44 /]# ansible --version
ansible 2.9.7
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  2 2020, 13:16:51) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
[root@a2a03b17bc44 /]# exit
exit
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
a2a03b17bc44        centos:ansible      "/bin/bash"         About a minute ago   Up About a minute                       ansible01
$ docker container stop ansible01
ansible01
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
$ 

{% endhighlight %}

The `--rm` switch will remove the container on ```docker container stop``` but if I want the container to remain even after I exit then I have to spin it up differently (minus the `--rm` switch).

{% highlight bash %}

$ docker run -t -d  --name ansible01 centos:ansible                                                                                                                                           
e455f69f1b134530494a0ad508146e3cd72a876c71b8ff39de5ff30acedc15dd

$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
e455f69f1b13        centos:ansible      "/bin/bash"         6 seconds ago       Up 5 seconds                            ansible01

$ docker exec -it ansible01 /bin/bash
[root@e455f69f1b13 /]# ansible --version
ansible 2.9.7
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  2 2020, 13:16:51) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
[root@e455f69f1b13 /]# exit
exit

$ docker container stop ansible01
ansible01

$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS                        PORTS               NAMES
e455f69f1b13        centos:ansible      "/bin/bash"         About a minute ago   Exited (137) 17 seconds ago                       ansible01

$ docker container start ansible01
ansible01

$ docker exec -it ansible01 /bin/bash
[root@e455f69f1b13 /]# datetimectl
[root@e455f69f1b13 /]# exit
exit
$ 

{% endhighlight %}

So I remeber Docker basics, I know I can have a php or NGINX container running and use `-p` to map the port, something like ```docker run -d -t -p 80:80 --name web01 centos:ansible```.

I will leave that for another post. 

