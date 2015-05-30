---
layout: post
title:  "MongoDB production cluster dev environment in Docker"
date:   2015-05-29 23:28:09
categories: docker mongo 
---
I've already covered [how to use docker to easily spin up a portable mongo instance][docker-mongo] but what if you need to emulate a [production cluster][production-cluster]?

Luckily we can use [skydock and skynet to handle DNS resolution][skydock] and take advantage of Docker's very low overhead and memory footprint to spin up our own "production" cluster that will only use the resources required by the actual mongo processes. (On my machine after following this guide, I'm using an additional 25 gigs of disk space and 1.2 gigs of ram.  Not too bad for 12 running containers!)

This guide is based mostly off of the [deploy a sharded cluster][sharded-cluster] tutorial from mongo but is customized to work with their [official docker images][docker-hub-mongo].

Before we go further, when you manually name containers, the names must be unique so if you want to stop and clear out all your running containers, here's a 1 liner to do it. WARNING, THIS WILL REMOVE ALL RUNNING CONTAINERS!!
{% highlight bash %}
docker ps | tail -n+2 | awk '{print $NF}' | xargs -I {} sh -c 'docker kill {} && docker rm {}'
{% endhighlight %}

First, we need to set up skydns and skydock. This allows for automagic dns config across all your docker containers.

Edit your /etc/init.d/docker file and set 

{% highlight bash %}
DOCKER_OPTS="--bip=172.17.42.1/16 --dns=172.17.42.1"
{% endhighlight %}

Then restart the docker daemon and start skydns and skydock.

{% highlight bash %}
docker run -d -p 172.17.42.1:53:53/udp --name skydns crosbymichael/skydns -nameserver 8.8.8.8:53 -domain docker
docker run -d -v /var/run/docker.sock:/docker.sock --name skydock crosbymichael/skydock -ttl 30 -environment dev -s /docker.sock -domain docker -name skydns
{% endhighlight %}

Next, we want to spin up 3 [configuration databases][configdb].

{% highlight bash %}
docker run -h 'config1.mongo.dev.docker' --name=config1 -d mongo sh -c 'mkdir /data/configdb; mongod --smallfiles --configsvr --port 27019'
docker run -h 'config2.mongo.dev.docker' --name=config2 -d mongo sh -c 'mkdir /data/configdb; mongod --smallfiles --configsvr --port 27019'
docker run -h 'config3.mongo.dev.docker' --name=config3 -d mongo sh -c 'mkdir /data/configdb; mongod --smallfiles --configsvr --port 27019'
{% endhighlight %}

You'll want to wait until all 3 are waiting for connections.  Here's a for loop that will print configN DONE when configN is done.  Once all 3 report done, you're ready for the next step.

{% highlight bash %}
for i in $(seq 1 3); do echo "config$i `docker logs config$i | grep 'waiting for connections'`" | sed 's/\(config[1-3]\).*/\1 DONE/g'; done
{% endhighlight %}

Next we need to spin up the [mongos][mongos] instance that is connected to our 3 configuration databases.

{% highlight bash %}
docker run -h 'mongos1.mongo.dev.docker' -d --name=mongos1 mongo sh -c 'mongos --configdb "config1.mongo.dev.docker:27019,config2.mongo.dev.docker:27019,config3.mongo.dev.docker:27019"'
{% endhighlight %}

Now we're ready to create our [replica sets][replication]!  Mongo's documentation recommends at least 2 replica sets to be added to the sharded cluster.  Replica sets must be an odd number of nodes and it seems like 3 is a recommended number.  This means we'll need 6 containers for our 2 replica sets.

{% highlight bash %}
docker run -h 'rs0node0.mongo.dev.docker' --name=rs0node0 -d mongo mongod --smallfiles --replSet rs0
docker run -h 'rs0node1.mongo.dev.docker' --name=rs0node1 -d mongo mongod --smallfiles --replSet rs0
docker run -h 'rs0node2.mongo.dev.docker' --name=rs0node2 -d mongo mongod --smallfiles --replSet rs0
docker run -h 'rs1node0.mongo.dev.docker' --name=rs1node0 -d mongo mongod --smallfiles --replSet rs1
docker run -h 'rs1node1.mongo.dev.docker' --name=rs1node1 -d mongo mongod --smallfiles --replSet rs1
docker run -h 'rs1node2.mongo.dev.docker' --name=rs1node2 -d mongo mongod --smallfiles --replSet rs1
{% endhighlight %}

All the nodes should be up now; at this point it's just a matter of configuration.

{% highlight bash %}
docker run -it --rm mongo sh -c 'mongo --host rs0node0.mongo.dev.docker --port 27017 --shell --eval "rs.initiate(); rs.conf(); rs.add(\"rs0node1.mongo.dev.docker\"); rs.add(\"rs0node2.mongo.dev.docker\");"'
{% endhighlight %}

Run rs.conf() and verify everything looks right, then exit with ctrl-D.

{% highlight bash %}
docker run -it --rm mongo sh -c 'mongo --host rs1node0.mongo.dev.docker --port 27017 --shell --eval "rs.initiate(); rs.conf(); rs.add(\"rs1node1.mongo.dev.docker\"); rs.add(\"rs1node2.mongo.dev.docker\");"'
{% endhighlight %}

Run rs.conf() and verify everything looks right, then exit with ctrl-D.

{% highlight bash %}
docker run -it --rm mongo sh -c 'mongo --host mongos1.mongo.dev.docker --port 27017 --shell --eval "sh.addShard(\"rs0/rs0node0.mongo.dev.docker:27017\");sh.addShard(\"rs1/rs1node0.mongo.dev.docker:27017\");"'
{% endhighlight %}

Run sh.status() and verify everything looks right, then exit with ctrl-D.

Now we just need to enable sharding for a given collection and load some data! I'm going to assume that you want to load the [enron dataset from my previous post][docker-mongo] but you can change this collection to whatever you want. After the first command run sh.status(), verify it looks right, and exit with ctrl-D.

{% highlight bash %}
docker run -it --rm mongo sh -c 'mongo --host mongos1.mongo.dev.docker --port 27017 --shell --eval "sh.enableSharding(\"enron_mail\"); sh.shardCollection(\"events.alerts\", { \"_id\": \"hashed\" } );"'
{% endhighlight %}

Run sh.status() and verify everything looks right, then exit with ctrl-D.

{% highlight bash %}
docker run -v ~/enron_dump/dump:/enron_dump -it --rm mongo sh -c 'mongorestore -h "mongos1.mongo.dev.docker" --port "27017" /enron_dump'
{% endhighlight %}

You've got a "production" sharded cluster running on your dev box!

[configdb]: http://docs.mongodb.org/manual/reference/config-database/
[docker-hub]: https://registry.hub.docker.com
[docker-hub-mongo]: https://registry.hub.docker.com/_/mongo/
[enron-site]: http://mongodb-enron-email.s3-website-us-east-1.amazonaws.com/
[enron-dump]: https://s3.amazonaws.com/mongodb-enron-email/enron_mongo.tar.bz2
[docker-mongo]: /docker/mongo/2015/03/20/docker-mongo-test-data.html
[docker-container-linking]: https://docs.docker.com/userguide/dockerlinks/
[production-cluster]: http://docs.mongodb.org/manual/core/sharded-cluster-architectures-production/
[replication]: http://docs.mongodb.org/manual/replication/
[sharded-cluster]: http://docs.mongodb.org/manual/tutorial/deploy-shard-cluster/
[skydock]: https://github.com/crosbymichael/skydock
[mongos]: http://docs.mongodb.org/manual/reference/program/mongos/#bin.mongos
