---
layout: post
title:  "Apache NiFi tls-toolkit single node standalone"
date:   2016-09-19 17:33:09
categories: tls nifi toolkit
---
Configuring TLS for Apache NiFi is a good way to ensure that traffic between users and NiFi as well as between NiFi cluster nodes is secure.  While a good security solution, TLS can be pretty tedious to configure.

To that end, NiFi now has the tls-toolkit which is intended to make that configuration much simpler and less error-prone.

The tls-toolkit has two main modes of operation:

1. Standalone utility in which it will generate a Certificate Authority, issue client and server certificates, and  output nifi.properties files for your nodes.
2. Client/Server mode in which the CA is a long running process listening for requests for certificates from clients.

The following will show a very simple usecase for a secure local NiFi instance.

First, download [Apache NiFi and the NiFi Toolkit archives.][nifidownload]

Unzip the archives into a folder:
{% highlight shell %}
mkdir toolkit-demo
cd toolkit-demo/
unzip ~/Downloads/nifi-toolkit-1.0.0-bin.zip
unzip ~/Downloads/nifi-1.0.0-bin.zip
{% endhighlight %}

Now generate a localhost NiFi configuration directory and client certificate:
{% highlight shell %}
cd nifi-toolkit-1.0.0/
bin/tls-toolkit.sh standalone -n localhost -C "CN=user, OU=NIFI"
{% endhighlight %}

Import the nifi-cert.pem into your browser as a trusted CA.

![Firefox -> Settings -> Advanced -> View Certificates]({{ site.baseurl }}/images/2016-09-16-tls-toolkit-intro-certs1.png)

![Import Nifi CA Cert]({{ site.baseurl }}/images/2016-09-16-tls-toolkit-intro-certs2.png)

Import The .p12 client certificate file into your browser as a client cert using the password in the generated .password file.

![Import Client Cert]({{ site.baseurl }}/images/2016-09-16-tls-toolkit-intro-certs3.png)

Copy the configuration for localhost into the client directory and cd into the NiFi directory.
{% highlight shell %}
cp localhost/* ../nifi-1.0.0/conf/
cd ../nifi-1.0.0/
{% endhighlight %}

Edit conf/authorizers.xml and set "Initial Admin Identity" property to be your user DN from before ex:
{% highlight xml %}
<property name="Initial Admin Identity">CN=user, OU=NIFI</property>
{% endhighlight %}

Start NiFi

{% highlight shell %}
bin/nifi.sh start
{% endhighlight %}

After NiFi finishes starting up, navigate to <https://localhost:9443/nifi/> in your browser.

It may prompt you for the key you would like it to use to authenticate but after you select it, you should see the following screen:

![Success]({{ site.baseurl }}/images/2016-09-16-tls-toolkit-intro-success.png)

Congratulations, you've used a self-signed CA to secure a NiFi instance!

[nifidownload]:https://nifi.apache.org/download.html
