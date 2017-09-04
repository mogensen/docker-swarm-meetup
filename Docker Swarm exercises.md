# Docker Swarm exercises

## Setup a Docker Swarm Cluster

In this part we use Play With Docker as a free provider of Alpine Linux Virtual Machine in the cloud. These can be used to build and run Docker containers and to play around with clusters in Swarm Mode.

This can be found at: [http://play-with-docker.com/](http://play-with-docker.com/)

If you are not at at all able to use vim which is the only editor avaible on play with docker you can install another.

    $ apk update
    $ apk add nano

Or install the [docker-machine driver for Play With Docker](https://github.com/play-with-docker/docker-machine-driver-pwd), to be able to edit files locally on your own machine.


### Create machine instances

Start by creating three machines using the "+Add New Instance" botton.

### Create cluster

Chose a machine to be the manager and initiate the Docker Swarm cluster on this machine.

    $ docker swarm init --advertise-addr eth0

On the output of this command we get the command that can be used to join workers to the cluster

    $ docker swarm join --token SWMTKN-1-2ypam... {ip}:2377

Run this command on the two other machines.
Now we can validate that we have a cluster by going to the manager note and run the following command.

    $ docker node ls
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    3kkk6vvth1gu5hi3v93q2har3 *   node1               Ready               Active              Leader
    207qifbk21o6h05afa4ypabk3     node2               Ready               Active
    dne037y5lvndl37kccj2028dk     node3               Ready               Active

## Deploy your first service

### Create a visualizer stack

To be able to see whats running on our cluster in a grahical interface we use the docker services called Visualizer.

Create a service stack with the visualizer container.

    docker service create --name=viz \
        -p=8081:8080 \
        --constraint=node.role==manager \
        --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
        dockersamples/visualizer:stable

Check the new endpoint on the manager node (port 8081) to see our cluster and the visualizer container.

#### Exercise

Delete the service we created above:

    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE                             PORTS
    rjvz7l1tzoal        viz                 replicated          1/1                 dockersamples/visualizer:stable   *:8081->8080/tcp
    [node2] (local) root@10.0.26.4 ~
    $ docker service rm viz

Change the command we just ran to a docker-compose.yml file. Looking something like this:

    version: "3"
    services:
      visualizer:
        image: {that image we want to use}
        ports:
          - {port stuff}
        volumes:
          - {volume mapping}
        deploy:
          placement:
            constraints:
             - {constraints}

[Docker Compose documentation](https://docs.docker.com/compose/compose-file/)

Redeploy the visualizer with the yaml file, and validate that it works!

    $ docker stack deploy --compose-file docker-compose.yml viz


## Create a complete Docker Swarm stack

We will now look at an example application called VoteingApp from one the Docker Labs.

![Architecture diagram](https://raw.githubusercontent.com/dockersamples/example-voting-app/master/architecture.png)

* A Python webapp which lets you vote between two options
* A Redis queue which collects new votes
* A .NET worker which consumes votes and stores them in…
* A Postgres database backed by a Docker volume
* A Node.js webapp which shows the results of the voting in real time

We can deploy the stack with this description.

    version: "3"
    services:
    
      redis:
        image: redis:alpine
        ports:
          - "6379"
    
      db:
        image: postgres:9.4
        volumes:
          - db-data:/var/lib/postgresql/data
    
      vote:
        image: dockersamples/examplevotingapp_vote:before
        ports:
          - 5000:80
        depends_on:
          - redis
    
      result:
        image: dockersamples/examplevotingapp_result:before
        ports:
          - 5001:80
        depends_on:
          - db
    
      worker:
        image: dockersamples/examplevotingapp_worker
    
    volumes:
      db-data:

First create a yml file with the data above and deploy it to our cluster as in the last exercise. Check that you can see all five containers in the visualizer. Also check that you can see the two frontends at port 5000 and 5001. Try voting.

### Scaling

News has spread like wildfire about our new awesome voting app and we need to have more servers avaible for voting.

Call the _docker service_ comandline with a _--help_ param to figure out how to scale the _vote_ service to 5 instances.

    $ docker service --help

Once you have scaled the service, check that you can see all five containers in the visualizer. Also chech that the vote page is now handled by different containers. The container handling a request is shown on the vote page.

Update the yml file with the new deployment info, stating that we want to have 5 replicas of the vote service.
[Docker Compose deploy documentation](https://docs.docker.com/compose/compose-file/#deploy)

### Network security

The next step in making our voting application more production ready is to make sure that the services only has access to what they need.

More specifically we want the services in the voting application to be on different networks. So that the frontend application vote only has access to the reddis cache and not to the postgress database or the result application.

_Frontend_
 - vote
 - redis
 - worker

_Backend_
 - worker
 - db
 - result

Create the two networks in the yml file and specify the networks on the services.

### Specify placement for the services

Next task on the production list is to make sure that many people voting doesn't kill our result database.
We want to make sure that this does not happen by putting the postgress database and the result application on the manager node and all the other on the worker nodes.

Update the yml file with conditions on all the services that specify where they are allowed to live. Validate by looking in the visualizer. Check that only the postgress and the result application is on the master.

### References
1. [Play With Docker on github](https://github.com/play-with-docker/play-with-docker)
2. [Deploy the Voting App on a Docker Swarm using Compose version 3](https://medium.com/lucjuggery/deploy-the-voting-apps-stack-on-a-docker-swarm-4390fd5eee4)
3. [Example voting app](https://github.com/dockersamples/example-voting-app)