# Miniswarm - Local docker swarm in one command

## What is Miniswarm?
Miniswarm is a tool that intends to make testing a [Docker swarm](https://docs.docker.com/engine/swarm/) cluster locally as easy as possible. miniswarm was inspired by [minikube](https://github.com/kubernetes/minikube) which does a similar thing for kubernetes clusters.

## Prereq

* Docker >= 1.12
* docker-machine >= 0.7.0
* Virtualbox

## Miniswarm and docker swarm healthchecks tutorial
I like to learn by doing, hopefully you do too. Lets install miniswarm, deploy our swarm cluster and some apps.

**install**
```
curl -sL miniswarm_url -o /usr/local/bin/miniswarm && chmod +rx /usr/local/bin/miniswarm
```

**start a cluster** - Pick your desired size
```
# 1 manger 2 workers
miniswarm start 3

# 1 manager cluster - if you want a smaller cluster
miniswarm start

# 2 mangers 3 workers - nice laptop or desktop :)
miniswarm start 2 3
```
A couple of minutes later, you should get this message
```
INFO: Stack starup complete. To connect to your stack, run the following command:
INFO: eval $(docker-machine env ms-manager0)
```

**visualize your cluster**

This will open a browser with a nice visualization of your docker swarm using [docker-swarm-visualizer](https://github.com/ManoMarks/docker-swarm-visualizer)
```
miniswarm vis
```

**Deploy our first service**

This service will be unhealthy due to failing [Goss](https://github.com/aelsabbahy/goss) healthchecks and missing dependencies. See next few steps for how we can debug and remedy this.
```
# Connect to our swarm
eval $(docker-machine env ms-manager0)

# Create a network for our service
docker network create healthyvote_net -d overlay

# Ensure network is set to driver=overlay, scope=swarm
docker network ls

# Create our first service
docker service create -p 8080:80 --replicas 2 --network healthyvote_net --name vote aelsabbahy/healthyvote
```

**Inspect the service health**
```
# This should show the service not running
docker service ls
docker service ps vote

# Lets look at the healthchecks using miniswarm
# -a shows all containers, including exited/failed containers
miniswarm health vote -a
```

We should see something like this:
```
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] ======
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Start: 2016-08-07 22:39:03.748704565 +0000 UTC
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] ======
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] .F
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Failures/Skipped:
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Title: Redis backend is reachable
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Meta:
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]     remedy.1: Deploy redis service if you haven't already
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]     remedy.2: ctrl-alt-delete
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]     remedy.3: take a nap aka human ctrl-alt-delete
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Addr: tcp://redis:6379: reachable:
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Expected
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]     <bool>: false
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] to equal
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]     <bool>: true
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q]
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Total Duration: 0.500s
[ms-worker0 vote.2.630mookf9pnc8dip5hl7yng3q] Count: 2, Failed: 1, Skipped: 0
```

**Deploy missing dependencies**

Lets remedy the healthcheck issue

```
docker service create --replicas 1 --network healthyvote_net --name redis redis

# Wait for it to show up in `miniswarm vis` or by using CLI
docker service ls

# Now that redis is deployed, lets if our vote service is running
docker service ls

# And the health..
miniswarm health vote
```

**Open service in browser**
```
# Open app in browser
miniswarm service vote

# print url, but don't open
miniswarm service vote --url
```

**View the logs**
```
miniswarm logs vote

# Tail the log file (-f has to be at the end for now)
miniswarm logs redis -f
```

**Delete our swarm cluster**
```
miniswarm delete
```

# FAQ
## Why the #$@^%$ is this written in Bash?

Two reasons:

1. I though it was going to be ~100 lines of bash, I was wrong.. very wrong :(.
2. I want users to be able to look at this script and see all the commands needed to set up a swarm cluster.
  * Go would be great for this tool, especially by leveraging the docker packages directly, but then the tool will be more of a blackbox to new users

## Why did you use Goss for healthchecks?
Mostly shameless self-promotion, and while we're on the topic, check out:
* [Goss](https://github.com/aelsabbahy/goss) - Project page
* [blog post](https://medium.com/@aelsabbahy/docker-1-12-kubernetes-simplified-health-checks-and-container-ordering-with-goss-fa8debbe676c) - On Using Goss with docker healthchecks and Kubernetes

## Why does this suck?

Because it's a quick hack I did over the weekend.. or I suck.. maybe both?

## I tried to use it and got an error

Open an issue, create a pull request.. contribute :)
