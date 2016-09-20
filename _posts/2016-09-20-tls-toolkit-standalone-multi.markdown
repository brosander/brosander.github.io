---
layout: post
title:  "Apache NiFi tls-toolkit multi-node standalone"
date:   2016-09-20 16:51:00
categories: nifi toolkit tls
---
The Apache NiFi tls-toolkit is an easier way to configure TLS for NiFi.  [Previously we used it to quickly configure a single secure NiFi instance.](/nifi/toolkit/tls/2016/09/19/tls-toolkit-intro.html)

Now we'll try a slightly more complicated usecase involving multiple secure NiFi nodes inside docker containers.

First, [install docker on your machine.](https://www.docker.com/products/docker)

Then, download [Apache NiFi and the NiFi Toolkit archives.][nifidownload]

Unzip the toolkit, generate the certificates, and extract the authorizers.xml from the archive:
{% highlight shell %}
mkdir toolkit-demo-2
cd toolkit-demo-2/
unzip ~/Downloads/nifi-toolkit-1.0.0-bin.zip
nifi-toolkit-1.0.0/bin/tls-toolkit.sh standalone -n 'nifi[1-3].nifi' -C 'CN=user, OU=NIFI'
unzip -p ~/Downloads/nifi-1.0.0-bin.zip nifi-1.0.0/conf/authorizers.xml > authorizers.xml
{% endhighlight %}

Update the authorizers xml with the following properties:
{% highlight xml %}
<property name="Initial Admin Identity">CN=user, OU=NIFI</property>
<property name="Node Identity 1">CN=nifi1.nifi, OU=NIFI</property>
<property name="Node Identity 2">CN=nifi2.nifi, OU=NIFI</property>
<property name="Node Identity 3">CN=nifi3.nifi, OU=NIFI</property>
{% endhighlight %}

Copy the authorizers to each instance's config directory:
{% highlight shell %}
find ./ -maxdepth 1 -name 'nifi*.nifi' -exec cp authorizers.xml {} \;
{% endhighlight %}

Generate an ssh key for use talking to the gateway:
{% highlight shell %}
mkdir nifi-ssh-keys
ssh-keygen -t rsa -b 4096 -f nifi-ssh-keys/id_rsa
{% endhighlight %}

Build the NiFi stack:
{% highlight shell %}
git clone https://github.com/brosander/dev-dockerfiles.git
dev-dockerfiles/nifi/ubuntu/buildStack.sh
{% endhighlight %}

Run the NiFi stack:
{% highlight shell %}
dev-dockerfiles/nifi/ubuntu/runStack.sh -p nifi-ssh-keys/id_rsa.pub -a ~/Downloads/nifi-1.0.0-bin.zip -c "`pwd`" -g -n 3
{% endhighlight %}

Wait until at least one NiFi instance comes up (You should see some Jetty output in the log, it may take a few minutes):
{% highlight shell %}
docker logs -f nifi1 2>/dev/null | grep -q "org.apache.nifi.web.server.JettyServer https://nifi1.nifi:9443/nifi"
{% endhighlight %}

SSH into the gateway at port 2001 and forward local port 1025 for use as a SOCKS proxy (this ip will be where docker exposes the port which may vary based on environment):
{% highlight shell %}
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i nifi-ssh-keys/id_rsa -p 2001 -D 1025 root@192.168.99.100
{% endhighlight %}

Configure your browser to use the ssh connection as a SOCKS proxy (for Firefox this is Settings -> Advanced -> Network -> Connection Settings)
![Success]({{ site.baseurl }}/images/2016-09-16-tls-toolkit-multi-socks.png)

Import the nifi-cert.pem into your browser as a trusted CA.

Import The .p12 client certificate file into your browser as a client cert using the password in the generated .password file.

After NiFi finishes starting up, navigate to one (or each) of the following in your browser.

1. <https://nifi1.nifi:9443/nifi/> 
2. <https://nifi2.nifi:9443/nifi/> 
3. <https://nifi3.nifi:9443/nifi/> 

It may prompt you for the key you would like it to use to authenticate but after you select it, you should see the following screen:

![Success]({{ site.baseurl }}/images/2016-09-16-tls-toolkit-multi-success.png)

Congratulations, you've used a self-signed CA to secure 3 node NiFi cluster in Docker!

[nifidownload]:https://nifi.apache.org/download.html
