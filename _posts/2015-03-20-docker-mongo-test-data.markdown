---
layout: post
title:  "MongoDB sample data in a Docker container"
date:   2015-03-20 11:28:09
categories: docker mongo 
---
Docker containers are a lightweight alternative to virtual machines.  They allow isolation of different applications so that they don't interfere with each other while avoiding the overhead of running a whole vm.  

Spinning up a prebuilt container is really easy once Docker is installed and configured.  There are a list of available images as well as instructions at [Docker Hub][docker-hub].

To spin up a mongo database instance, run:
{% highlight bash %}
docker run --name my-mongo-instance -d mongo
{% endhighlight %}

If this is the first time you've run the mongo container, Docker will automatically pull down the relevant layers before starting the container.  The diff-based filesystem allows images to share common bases to minimize download size and disk usage.

Now that your mongo instance is running inside its own container, you'll want to connect to it to load data.  There are a few options here.  You can use another mongo container to connect with the mongo command line client:

{% highlight bash %}
docker run -it --link my-mongo-instance:mongo --rm mongo sh -c 'exec mongo "$MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT/test"'
{% endhighlight %}

If you have a data dump you'd like to load up, you can do so with the mongorestore command. (This writeup will use the [Enron email dumps][enron-site] available for download as a [tar.bz2][enron-dump])

After extracting the archive, you can mount it into the container with the -v flag on the docker run command:

{% highlight bash %}
cd ~/ && mkdir enron_dump && cd enron_dump
tar -jxf ~/Downloads/enron_mongo.tar.bz2
docker run -v ~/enron_dump/dump:/enron_dump -it --link my-mongo-instance:mongo --rm mongo sh -c 'exec mongorestore -h "$MONGO_PORT_27017_TCP_ADDR" --port "$MONGO_PORT_27017_TCP_PORT" /enron_dump'
{% endhighlight %}

Now that the data is loaded, you can link to your mongo container with another docker container or connect to it with a tool running normally via the container's ip address at port 27017.

{% highlight bash %}
{% raw %}
docker inspect -f="{{.NetworkSettings.IPAddress}}" my-mongo-instance
{% endraw %}
{% endhighlight %}

When you're done with your container, you can stop it using docker stop:

{% highlight bash %}
docker stop my-mongo-instance
{% endhighlight %}

If you want to restart it later, use docker start:

{% highlight bash %}
docker start my-mongo-instance
{% endhighlight %}

You can even turn your container into an image to base other containers on:

{% highlight bash %}
docker commit my-mongo-instance
{% endhighlight %}

If you'd like to name it, you can now tag the commit id printed by the previous step (substitute your id in below):

{% highlight bash %}
docker tag COMMIT_ID_FROM_PREV_STEP my-mongo-image
{% endhighlight %}

my-mongo-image should now show up in your image list:

{% highlight bash %}
docker images
{% endhighlight %}

You can even start another container based on this image:

{% highlight bash %}
docker run --name my-mongo-instance-custom -d my-mongo-image
{% endhighlight %}

And connect to it from the shell, load up with data, use it from an application, etc as described above.

[docker-hub]: https://registry.hub.docker.com
[enron-site]: http://mongodb-enron-email.s3-website-us-east-1.amazonaws.com/
[enron-dump]: https://s3.amazonaws.com/mongodb-enron-email/enron_mongo.tar.bz2
