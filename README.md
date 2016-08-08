# Miniswarm - Docker Swarm cluster in one command

## What is Miniswarm?
Miniswarm is a tool that intends to make creating and managing a local [Docker Swarm](https://docs.docker.com/engine/swarm/) cluster as easy as possible. Miniswarm was inspired by [Minikube](https://github.com/kubernetes/minikube) which does a similar thing for kubernetes clusters. See FAQ below for info on managing a remote Swarm cluster.

The tool takes less than 10 minutes to learn, see the tutorial section below, or watch this tutorial video:
* [Miniswarm: Docker Swarm Tutorial](https://youtu.be/in1ItGKDr98)

## Prerequisites

* Docker >= 1.12
* docker-machine >= 0.7.0
* VirtualBox - Needed for local cluster, see FAQ for remote Swarm cluster

## Miniswarm and Docker Swarm healthchecks tutorial (less than 10min)
In this tutorial we'll install miniswarm, create a Swarm cluster, deploy some apps and learn all the features of miniswarm in the process.

**Install**
```
# As root
curl -sSL https://raw.githubusercontent.com/aelsabbahy/miniswarm/master/miniswarm -o /usr/local/bin/miniswarm
chmod +rx /usr/local/bin/miniswarm
```

**Start a cluster - Pick your desired size**
```
# 1 manager 2 workers
miniswarm start 3

# 1 manager cluster - if you want a smaller cluster
miniswarm start

# 2 managers 3 workers - nice laptop or desktop :)
miniswarm start 2 3
```
A couple of minutes later, you should get this message
```
INFO: Stack starup complete. To connect to your stack, run the following command:
INFO: eval $(docker-machine env ms-manager0)
```

**Visualize your cluster**

This will open a browser with a nice visualization of your Docker Swarm using [docker-swarm-visualizer](https://github.com/ManoMarks/docker-swarm-visualizer)
```
miniswarm vis
```

**Deploy your first service - This will initially be failing to showcase healtchecks**

This service will be unhealthy due to failing [Goss](https://github.com/aelsabbahy/goss) healthchecks and missing dependencies. See next few steps for how we can debug and remedy this.
```
# Connect to your cluster
eval $(docker-machine env ms-manager0)

# Create a network for your service
docker network create healthyvote_net -d overlay

# Ensure network is set to driver=overlay, scope=swarm
docker network ls

# Create your first service
docker service create -p 8080:80 --replicas 2 --network healthyvote_net --name vote aelsabbahy/healthyvote
```

**Inspect the service health**
```
# Wait for the service to finish preparing, but it won't ever be ready due to failing health
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
docker service ps redis

# Now that redis is deployed, the  vote service should now be running
docker service ls

# And the health..
miniswarm health vote
```

**Open service in browser**
```
# Open app in browser
miniswarm service vote

# print url, but don't open (--url has to be at the end for now)
miniswarm service vote --url
```

**View the logs**
```
miniswarm logs vote

# Tail the log file (-f has to be at the end for now)
miniswarm logs vote -f
```

**Scale down your cluster**
```
# Take a look at `miniswarm vis` GUI to see the services move around as we scale down
miniswarm scale 2
```

**Delete your cluster**
```
miniswarm delete
```

# FAQ
## Can this be used to manage a remote Swarm cluster?
The tool was written with local Swarm cluster in mind. That said, it can probably be used to manage a remote Swarm clusters, but that hasn't been tested. Take a look at MS_CREATE variable at the top of the script. Feel free to submit a pull-request to improve this or add more support.

## How do I disable the color output?

```
MS_NOCOLOR=1 miniswarm ..

# or
export MS_NOCOLOR=1
miniswarm ...
```

## Does this work on Mac/OSX?
It should, but I don't own a Mac, so I depend on others to verify it. So.. if something is broke on mac, please submit a pull request.

## Why did you use Goss for healthchecks?
Mostly shameless self-promotion, and while we're on the topic, check out:
* [Goss](https://github.com/aelsabbahy/goss) - Project page
* [blog post](https://medium.com/@aelsabbahy/docker-1-12-kubernetes-simplified-health-checks-and-container-ordering-with-goss-fa8debbe676c) - Using Goss with Docker healthchecks and Kubernetes

## Why the #$@^%$ is this written in Bash?

Two reasons:

1. I though it was going to be ~100 lines of bash, I was wrong.. very wrong :(.
2. I want users to be able to look at this script and see all the commands needed to set up a Swarm cluster.
  * Go would be great for this tool, especially by leveraging the Docker go packages directly, but then the tool will be more of a blackbox to new users

## I'm getting intermittent network issues, why is this happening?
Honestly, I don't know.. I see them too. Either I'm doing something dumb, or Docker Swarm mode has intermittent DNS issues. Hopefully with more users we can get to the bottom of this.

## Why is healthyvote not a Docker automated build?
Because Dockerhub doesn't support HEALTHCHECK in Dockerfile yet. The code for this image can be found in the [healthyvote/](https://github.com/aelsabbahy/miniswarm/tree/master/healthyvote) folder.

## Why does this suck?

Because it's a quick hack I did over the weekend.. or I suck.. maybe both?

## I tried to use it and got an error

Open an issue, create a pull request.. contribute! :)

## Why is the CLI parsing so bad?

See the last two questions.
