---
layout: post
title:  "Total Recall Docker 101"
date:   2020-05-03 07:52:5
categories: [blog]
tags: [docker]
excerpt_separator: <!--more-->
---

Of late, Docker and Ansible have caught my fancy. I have worked in limited capacity with both the tools, mostly doing PoC work.
Today, I thought of running a container, a simple container, just to asses if I still remembered the commands to run a container.

{% highlight bash %}

$ docker run -t -d --rm centos # forgot to give it a name
$ docker run -t d --rm --name web01
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
f859762bcba1        centos              "/bin/bash"         9 seconds ago       Up 8 seconds                            web01

{% endhighlight %}
<!--more-->

OK, so I remembered how to run a container. I also remembered that if I do a ```docker container stop web01``` the container will disappear.

Do I remember how to run a container & interact with it?

{% highlight bash %}

$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
f859762bcba1        centos              "/bin/bash"         9 seconds ago       Up 8 seconds                            web01
zterraform:~/environment $ docker exec -it web01 /bin/bash
[root@f859762bcba1 /]# cat /etc/os-release 
NAME="CentOS Linux"
VERSION="8 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="8"
PLATFORM_ID="platform:el8"
PRETTY_NAME="CentOS Linux 8 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:8"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-8"
CENTOS_MANTISBT_PROJECT_VERSION="8"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="8"

[root@f859762bcba1 /]# 

{% endhighlight %}

Huh! not bad, I remeber the basics. Can I spin, say 3 containers from command line?

{% highlight bash %}

$ for i in {1..3}; do docker run -d -t --rm --name web$i centos; done                                                                                                                         
a99d23d46dd4d997d759568abb9dcd7c0ab6f98e94d40dca55bcba8c66ceaafd
cce72443546334febb806ceada1aafb28d18103a2e3da44fea07a1a00af2e9cd
9c564110094c80f010ced2bccddc0d94ca8c16059c527297bfb42ea1c7638a27
$ docker container ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
9c564110094c        centos              "/bin/bash"         4 seconds ago       Up 3 seconds                            web3
cce724435463        centos              "/bin/bash"         4 seconds ago       Up 3 seconds                            web2
a99d23d46dd4        centos              "/bin/bash"         5 seconds ago       Up 4 seconds                            web1
{% endhighlight %}

Not bad, eh! 


{% highlight bash %}

$ docker container stop $(docker container ls -q)

{% endhighlight %}

