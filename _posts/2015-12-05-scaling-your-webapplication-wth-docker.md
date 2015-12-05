---
layout: post
title:  "Scaling your webapplication with Docker"
date: 2015-12-05 18:00:00 +0100
categories: 
 - Docker
 - consul
tags:
 - Docker
 - consul 
commentIssueId: 1
---

**Imagine, the webapplication you developed has suddenly become quite popular and you need to scale up. In the old days this required installing another virtual 
machine, which takes loads of time. Now, with the solution provided here, scaling up and down can be done in an instant. In this article I will show you how you 
can build an highly scalable environment for your webapps using Docker containers.**

{::options parse_block_html="true" /}
<div class="toc">
<h4>Contents</h4>
* toc
{:toc}
</div>

[Docker](https://www.docker.com/) is an open source tool for builing and running containers. Containers are an all-in-one package filled the application and its dependencies.
The benefit of using containers is that it runs directly on the operating system, which saves you from a virtualization-layer. Every container runs
seperately, so your environment will be multi-tennnant safe.

##Tooling
- Docker: In the current IT environment there is never enough speed, VM's are slow and containers are hot. Docker is currently the most popular 
container software.
- Docker-compose: tool for defining and launching multi-container applications.
- Haproxy: Open source load balancer for TCP and HTTP based services
- Consul: A tool for service discovery and configuration
- Apache with php: Just an example of an webapplication, but you can use any other webapplication type
- Supervisord: As we run next to the primary process a consul application in the container we need something to launch them

Of the list above you only need to install the Docker toolbox. You can find it at the Docker [website](https://www.docker.com/docker-toolbox). Make sure you 
have the docker "default" virtual machine created by opening Docker Kitematic or use docker-machine create to create

##Overview
In the image below you can see how the environment will be. As you can see I use three different parts of consul. The agent, the server and template. 
The agent tells the server it is online and which services the machine/container has available. Consul template queries the server and replaces the 
configuration file of haproxy which it also notifies. On shutting down a webserver the consul agent notifies the server it is leaving and consul template 
will remove the server from the configuration file. 

![container overview](/images/2015-12-01-scaling-your-dockerized-webapplication/overview.png)

##Docker containers##
As mentioned before I am using Docker containers in this setup. Three different containers can be identified:

- Consul server
- Haproxy with consul-template
- Webservice with consul agent

First of all, get the git repository with the example code:
{% highlight bash %}
git clone https://github.com/MansM/docker-scalingdemo.git
{% endhighlight %}

###Consul server
Change to the folder in your terminal or windows commandline. The consul server is the consul-server container made by gliderlabs, to start it:

{% highlight bash %}
docker run --rm --name consul -p 8500:8500 gliderlabs/consul-server -bootstrap
{% endhighlight %}

to view the consul server ui, you can browse to the Docker_ip:8500, you can locate the ip with running the command:
{% highlight bash %}
docker-machine ip default 
{% endhighlight %}
(if it default is not the correct Docker-machine, find the correct one with Docker-machine ls)

You should now see something like:
![Consul ui](/images/2015-12-01-scaling-your-dockerized-webapplication/consul-ui.png)

###Web application
Now its time to add a webserver to the pool, first of all we need to build and run the Docker container

{% highlight bash %}
docker build -t apachephp apachephp
docker run  --rm --name web -p 8080:80 --link="consul" apachephp
{% endhighlight %}
When you refresh the consul-ui screen, you will see the webserver has been added. 
![Consul ui with webserver](/images/2015-12-01-scaling-your-dockerized-webapplication/consul-ui-web.png)

####Dockerfile
The main item of any Docker container is the Dockerfile, this contains the building steps from which Docker builds the image.
In this case we are using as base image the image from php including apache (line 1). On line 4 we install supervisord and two
utilities to get the rest of the Dockerfile working. The consul agent is downloaded and installed on line 9. The configuration for 
supervisord and consul is copied into the image on 12 and 13. The last line is the command that is executed on launching the container, 
here it is starting supervisord to launch the other processes.

{% highlight docker linenos %}
FROM php:5.6-apache
ENV CONSUL_AGENT_VERSION=0.5.2

RUN apt-get update && apt-get install -y supervisor wget unzip
RUN apt-get clean

RUN mkdir -p /var/log/supervisor
RUN mkdir -p /etc/consul.d 
RUN wget https://releases.hashicorp.com/consul/${CONSUL_AGENT_VERSION}/consul_${CONSUL_AGENT_VERSION}_linux_amd64.zip -O /tmp/consul-agent.zip && unzip /tmp/consul-agent.zip && mv consul /usr/local/bin

COPY src/ /var/www/html/
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY web.json /etc/consul.d/web.json

CMD ["/usr/bin/supervisord"]
{% endhighlight %}


####web.json
This file is the configuration or the consul agent. The service web is the selector which later will be used to identify the needed containers. Line 7 is quite important, it
makes the agent notify the server it is leaving when the containter is stopped. This will save you from deregistering every stopped node in the consul ui.
{% highlight json linenos %}
{
  "service": {
    "name": "web",
    "tags": ["php"],
    "port": 80
  },
  "leave_on_terminate": true
}
{% endhighlight %}

####supervisord.conf
In the configuration file for supervisord we mention that supervisord shouldnt be started as a deamon. This will make the output be shown directly (and retrievable
with Docker logs).
{% highlight ini linenos %}
[supervisord]
nodaemon=true

[program:apache]
command=/usr/local/bin/apache2-foreground

[program:consulagent]
command=consul agent -data-dir=/tmp/consul -join consul -config-dir /etc/consul.d
{% endhighlight %}

###Load balancer
The final container in the setup is the load balancer. You can start it with the following commands:
{% highlight bash linenos %}
Docker build -t loadbalancer haproxy
Docker run --rm -p 80:80 -p 9000:9000 --link="consul" --link="web" --name loadbalancer loadbalancer
{% endhighlight %}

The load balancer exists of two main components: consul template and haproxy. I have chosen haproxy in this setup so I can switch to non http backends as well.
In your browser go to Docker_ip:9000/haproxy_stats (User/Pass: admin) to see the statistics page of haproxy, which shows 1 backend webserver.
![haproxy](/images/2015-12-01-scaling-your-dockerized-webapplication/haproxy.png)

####Dockerfile
The Dockerfile of the loadbalancer consists of installing the software (line 5 & 6) and copying configuration files into the image (line 9 & 10).
{% highlight Docker linenos %}
FROM debian:latest
MAINTAINER Mans Matulewicz
ENV CONSUL_TEMPLATE_VERSION=0.10.0

RUN apt-get update && apt-get install -y haproxy wget unzip
RUN ( wget --no-check-certificate https://github.com/hashicorp/consul-template/releases/download/v${CONSUL_TEMPLATE_VERSION}/consul-template_${CONSUL_TEMPLATE_VERSION}_linux_amd64.tar.gz -O /tmp/consul_template.tar.gz && gunzip /tmp/consul_template.tar.gz && cd /tmp && tar xf /tmp/consul_template.tar && cd /tmp/consul-template* && mv consul-template /usr/bin && rm -rf /tmp/* )
RUN apt-get clean

COPY haproxy.ctmpl /etc/haproxy.ctmpl
COPY haproxy.json /etc/haproxy.json

EXPOSE 80/tcp 9000/tcp
CMD ["consul-template", "-config=/etc/haproxy.json", "-consul=consul:8500"]
{% endhighlight %}
####haproxy.ctmpl
Below is only a part of the haproxy config, the part that is modified by consul-template. At line 7 the loop starts for every node that delivers the service web and it will create 
a line for every available server. The check what is normally found at the end of the haproxy config for every node is not used, this because we only have working nodes available.
{% highlight nginx linenos %}
{% raw %}
backend nodes
    mode http
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    balance roundrobin {{range service "web"}}
    server {{.Node}} {{.Address}}:{{.Port}}{{end}}{% endraw %}
{% endhighlight %}
####haproxy.json
This is the configuration for consul template. Using the source template file, it will generate the destination configuration file and will invoke haproxy with the newly created
configuration file
{% highlight javascript linenos %}
template {
  source = "/etc/haproxy.ctmpl"
  destination = "/etc/haproxy/haproxy.cfg"
  command = "haproxy -f /etc/haproxy/haproxy.cfg -sf $(pidof haproxy) &"
}
{% endhighlight %}

##Docker-compose
Until now we have started all the containers one by one, but there is an more easy way todo it. We will using Docker-compose, it is shipped with the Docker toolbox. 
Before we can use it we have to make sure all the manually started containers are gone, otherwise you will get conflicts with ports. Below you can see the the running containers and 
the command to stop them.
{% highlight javascript bash %}
Manss-MacBook-Air:~ Mans$ Docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                                                                NAMES
c475f80693b9        loadbalancer               "consul-template -con"   3 seconds ago       Up 3 seconds        0.0.0.0:80->80/tcp, 0.0.0.0:9000->9000/tcp                                           loadbalancer
ffd06220716f        apachephp                  "/usr/bin/supervisord"   17 seconds ago      Up 17 seconds       0.0.0.0:8080->80/tcp                                                                 web
16df4e7a5e64        gliderlabs/consul-server   "/bin/consul agent -s"   29 seconds ago      Up 28 seconds       8300-8302/tcp, 8400/tcp, 8301-8302/udp, 8600/tcp, 8600/udp, 0.0.0.0:8500->8500/tcp   consul
Manss-MacBook-Air:~ Mans$ Docker stop loadbalancer web consul
loadbalancer
web
consul
{% endhighlight %}

To start all the containers of this project, use the command Docker-compose up. It will start all the containers and show the logs.
Now it's time for the stuff its all about: scaling! To increase the amount of webservers to 5, you just have to give this command:
{% highlight javascript bash %}
Manss-MacBook-Air:scalingdemo Mans$ Docker-compose scale web=5
Creating and starting 2 ... done
Creating and starting 3 ... done
Creating and starting 4 ... done
Creating and starting 5 ... done
{% endhighlight %}
Within a couple of seconds you are now running your webapplication on 5 webservers: 
![Consul ui upscaled](/images/2015-12-01-scaling-your-dockerized-webapplication/consul-ui-upscaled.png)
![haproxy upscaled](/images/2015-12-01-scaling-your-dockerized-webapplication/haproxy-upscaled.png)

###Docker-compose.yml
All the magic of the Docker-compose is configured in the Docker-compose.yml listed below. In essence it is just listing all the containers and the runtime 
parameters.
{% highlight javascript yaml %}
consul:
  image: gliderlabs/consul-server
  command: -bootstrap
  ports:
   - "8500:8500"
loadbalancer:
  build: haproxy
  ports:
   - "80:80"
   - "9000:9000"
  links:
   - web
   - consul
web:
  build: apachephp
  links:
    - consul
{% endhighlight %}
##Conclusion
Creating an highly scalable environment is not hard as this blogpost will have shown you. It is usable for containers as well for virtual machines.
In a later post I will add Docker swarm to the mix and extend the functionality over multiple (Docker-)machines.

##Sources

- http://sirile.github.io/2015/05/18/using-haproxy-and-consul-for-dynamic-service-discovery-on-docker.html
- http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-Docker-compose-and-consul-part-14-a-taste-of-what-is-to-come/