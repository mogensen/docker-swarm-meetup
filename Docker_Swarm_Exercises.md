# Docker Swarm exercises

## About this exercise set

> To help the users of this exercise to learn how to use docker and where to get help when they are stuck in the real world, I have tried to create exercieses that forces the user to look in documentation and experiment.
> **This means, that the excercise set is not a simple copy & paste exercise**

## Setup a Docker Swarm Cluster

In this part we use Play With Docker as a free provider of Alpine Linux Virtual MachineS in the cloud. These can be used to build and run Docker containers and to play around with clusters in Swarm Mode.

This can be found at: [http://play-with-docker.com/](http://play-with-docker.com/)

If you are not at at all able to use vim, which is the only editor avaible on play with docker, you can install another.

    $ apk update
    $ apk add nano

Or install the [docker-machine driver for Play With Docker](https://github.com/play-with-docker/docker-machine-driver-pwd), to be able to edit files locally on your own machine.

### Create machine instances

Start by creating three machines using the "+Add New Instance" botton.

### Create cluster

Chose a machine to be the manager and initiate the Docker Swarm cluster on this machine.

    $ docker swarm init --advertise-addr eth0

In the output of this command, we get the command that can be used to join workers to the cluster

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

Delete the service we created above.

    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE                             PORTS
    rjvz7l1tzoal        viz                 replicated          1/1                 dockersamples/visualizer:stable   *:8081->8080/tcp
    
    $ docker service rm viz

Change the command we just ran to a docker-compose file with the name _docker-visualize.yml_. Looking something like this:
(Note that the volume mapping does not have quite the same format in the commandline as it does in the docker-compose file)

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

    $ docker stack deploy --compose-file docker-visualize.yml viz

**[Note]** This command is used BOTH for deploying a stack first time and for updating the desired state of the stack. 

## Create a complete Docker Swarm stack

We will now look at an example application called VotingApp from one the Docker Labs.

![Architecture diagram](https://raw.githubusercontent.com/dockersamples/example-voting-app/master/architecture.png)

* A Python webapp which lets you vote between two options
* A Redis queue which collects new votes
* A .NET worker which consumes votes and stores them in...
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
        depends_on:
          - db
          - redis
    
    volumes:
      db-data:

Start by creating a yml file with the data above and deploy it to our cluster as in the last exercise.
Check that you can see all five containers in the visualizer. You should also check that you can see the two frontends at port 5000 and 5001.
Try voting.

## Scaling the frontend service

News has spread like wildfire about our new awesome voting app and we need to have more services available for voting.

#### Exercise

Call the _docker service_ commandline with a _--help_ param to figure out how to scale the _vote_ service to 5 instances.

    $ docker service --help

Once you have scaled the service:

 - Check that you can see all five containers in the visualizer.
 - Check that the vote page is now handled by different containers.
     * The container handling a request is shown on the vote page.

#### Exercise

Update the yml file with the new deployment info, stating that we want to have 5 replicas of the vote service.
[Docker Compose deploy documentation](https://docs.docker.com/compose/compose-file/#deploy)

## Network security

The next step in making our voting application more production ready is to make sure that the services only has access to what they need.

More specifically, we want the services in the voting application to be on different networks. So that the frontend application _vote_ only has access to the redis cache and not to the postgres database or the result application.

We also need the _worker_ service to have access to both networks, because it is the one responsible for moving votes from the redis cache to the postgres database.

| _Frontend_    | _Backend_     |
| ------------- |---------------|
| vote          | worker        |
| redis         | db            |
| worker        | result        |

#### Exercise

Create the two networks in the yml file and specify the networks on the services.
Note that this is something that docker has changed a lot over time.
Newest documentation is here: [Docker compose endpoint mode](https://docs.docker.com/compose/compose-file/#endpoint_mode)

## Playing with node failure

To simulate a failing node (bloody hardware failing all the time) try deleting one of the worker nodes in the Play With Docker interface.
When doing this keep an eye on the visualizer and see how all the containers are redeployed on the remining node.

Try creating two new nodes and joining them to the cluster. By default docker swarm will not move running containers to the new nodes.
Try scaling up the _vote_ or _worker_ services and see how the containers are placed.

## Specify placement for the services

Next task on the production list is to make sure that our result database are only running on machines that have ssd.
Add a label called _disk_ to all our worker nodes, with _disk=hdd_ for two of them and _disk=ssd_ on the last worker
See [documentation](https://docs.docker.com/engine/swarm/manage-nodes/#add-or-remove-label-metadata) for adding lables.

#### Exercise

Update the yml file with conditions on the _db_ service that specify that it can only be deployed on _disk=ssd_. Validate by looking in the visualizer.

You can also change the other services to make sure that only the database runs on machines with ssd.

Try setting the manager as _drained_ so no containers are scheduled on the manager.

### References
1. [Play With Docker on github](https://github.com/play-with-docker/play-with-docker)
2. [Deploy the Voting App on a Docker Swarm using Compose version 3](https://medium.com/lucjuggery/deploy-the-voting-apps-stack-on-a-docker-swarm-4390fd5eee4)
3. [Example voting app](https://github.com/dockersamples/example-voting-app)
