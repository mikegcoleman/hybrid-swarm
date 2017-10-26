# Deploying Hybrid Applications with Docker Swarm



## What is "Orchestration"

If you heard about containers you've probably heard about orchestration as well. Container orchestrators, such as Docker Swarm and Kubernetes, provide a ton of functionality around managing and deploying containerized applications. But the two most fundamental things they do are clustering and scheduling (that's not all they do by a long shot, but they are arguably the two most important fuctions).

Clustering is the concept of taking a group of machines and treating them as a single computing resource. These machines are capable of accepting any workload becaues they all offer the same capabilities. These clustered machines don't have to be running on the same infrastructure - they could be a  mix of bare metal and VMs for instance.

Scheduling is the process of deciding where a workload should execute. When an admin starts a new instance of website she can decide what region it needs to go on or if it should be on bare metal or in the cloud. The scheduler will make that happen. Schedulers also make sure that the application maintains its desired state. For example, if there were 10 copies of a web site running, and one of them crashed, the scheduler would know this and start up a new instance to take the failed one's place.

Docker features a built-in orchestrator: Docker Swarm. It provides both clustering and scheduling as well as many other advanced services.

This lab will start with the deployment of a 3 node Docker Swarm cluster.

## Build your cluster

1. In your browser navigate to [Play with Docker](https://dockercon.play-with-docker.com)

1. In the PWD interface click `+ Add new instance` to instantiate a linux node

1. Repeat step 1 to add a second node to the cluster.

1. Click the `Windows containers` slider and then click `+ Add New Instance`

    There are now three standalone Docker hosts: 2 linux and 1 Windows. 

1. In the console for `node1` initialize Docker Swarm

    ```bash
    $ docker swarm init --advertise-addr eth0

    Swarm initialized: current node (ujsz5fd1ozr410x1ywuix7sik) is now a manager.

    To add a worker to this swarm, run the following command:

        docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
    ```

    `node1` is now a Swarm manager node. Manager nodes are responsible for ensuring the integrity of the cluster as well as managing running services. 

1. Copy the `docker swarm join` output from `node1`

1. Switch to `node2` and paste in the command. **Be sure to copy the output from your PWD window, not from this lab guide**

    ```bash
    $ docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377

    This node joined a swarm as a worker.
    ```
1. Switch to the Windows node and paste the same command at the Powershell prompt

    ```bash
    PS C:\> docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377

    This node joined a swarm as a worker.
    ```

    The three nodes have now been clustered into a single Docker swarm. An important thing to note is that clusters can be made up of Linux nodes, Windows nodes, or a combination of both.

1. Switch back to `node1`

1. List the nodes in the cluster

    ```bash
    $ docker node ls

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    uqqgsvc1ykkgoamaatcun89wt *   node1               Ready               Active              Leader
    g4demzyr7sd2ngpa8wntbo0l4     node2               Ready               Active
    xflngp99u1r9pn7bryqbbrrvq     win000046           Ready               Active
    ```

1. Commands against the swarm can only be issued from the manager node. Attempting to run the above command from the `Windows` node.

    ```bash
    PS C:\ docker node ls

    Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
    ```

1. Move back to `node1` and promote the Windows node to a manager role:

    > **Note** Be sure to use the name of your Windows node

    ```bash
    $ docker node promote winxxxxx

    Node winxxxxx promoted to a manager in the swarm.
    ```

1. Move back to the `Windows` node and run the `docker node ls` command again

    ```bash
    PS C:\ docker node ls

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    ofetnt63m2f6uthres6jnne08 *   win000198           Ready               Active              Reachable
    q8swkmaldrhm2abacbzafscbo     node1               Ready               Active              Leader
    ymj4gg6bnfvgc70j8r9zef593     node2               Ready               Active
    ```

1. Move back to the `node1` node and demote the Windows node back to a worker.

    ```bash
    $ docker node demote winxxxxx

    Manager winxxxx demoted in the swarm.
    ```

## Deploying Swarm services

While we use `docker run` to instantiate docker containers on a single non-clustered node, the preferred way to actually run applications in a Swarm cluster is via a Docker [*service*](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/).

Services are an abstraction in Docker that represent an application or component of an application. For instance a web front end service connecting to a database backend service. You can deploy an application made up of a single service. In fact, this is quite common when [modernizing traditional applications](https://goto.docker.com/rs/929-FJL-178/images/SB_MTA_04.14.2017.pdf). 

The service construct provides a host of useful features including:

* Load balancing
* Layer 4 and layer 7 routing meshes
* Desired state reconciliation
* Service discovery
* Healthchecks
* Upgrades and rollback
* Scaling

This workshop cannot possibly cover all these topics, but we will cover several of them. 

1. Let's start by creating our first service:

    ```bash
    docker service create \
    --name hostname \
    --replicas 1 \
    --publish 8080:8080 \
    --detach=true \
    dockersamples/linux-hostname:latest

    t5aqr8z0prphzidnd44dhojpb
    ```

    Let's look at each line:

    * `docker service create`: Create a new Swarm service
    * `--name hostname`: Names the service hostname
    * `--replicas 1`: Start a single task under the service
    * `--publish 8080:8080`: Route any requests coming into the swarm cluster on port 8080 to the service at port 8080
    * `--detach=true`: Run the service in the background
    * `dockersamples\linux-hostname:latest`: Base our service on the `dockersamples\linux-hostname` image

1. Check to ensure that your service was created. `docker service ls` will list all the services running on a Swarm cluster. 

    ```bash
    $ docker service ls

    ID                  NAME                MODE                REPLICAS            IMAGE                      PORTS
    pypc8c8jh1zm        hostname            replicated          1/1                 dockersamples/linux-hostname:latest   *:8080->8080/tcp
    ```

1. The `docker service ls` command shows us that the service has been created and that 1 of the expected 1 (`1/1`) tasks have been started, but it doesn't give us any detail on that task. `docker ps` shows the tasks running as part of a given service.

    ```bash
    $ docker service ps hostname

    ID                  NAME                IMAGE                                 NODE  DESIRED STATE       CURRENT STATE            ERROR               PORTS
    x47pe8ov8wro        hostname.1          dockersamples/linux-hostname:latest   node1  Running             Running 43 seconds ago
    ````

    The output of the command displays the image the task is based on, the node where it is executing, the desired state of the task, and the actual state of task.

    In the example above notice the task is running on `node1`

    > Note: Your task may be running on `node2` - be sure to make a note of where it's actually running.

1. This service simply returns the host name of the container that responds to the request. **From the node where the taks is running** issue a `curl` command to see the output.

    > Note: If the output from `docker service ps` showed your task running on `node2` be sure to switch to `node2`

    ```bash
    $ curl localhost:8080

    Served by host: b738b7709d4b
    ```
1. Check to see the running containers on that host:

    ```bash
    $ docker container ps 

    CONTAINER ID        IMAGE                                 COMMAND             CREATED             STATUS              PORTS         NAMES
    b738b7709d4b        dockersamples/linux-hostname:latest   "/hello-go"         6 minutes ago       Up 6 minutes        8080/tcp      hostname.1.x47pe8ov8wrov69zv3zajho27
    ```

    Notice that the `CONTAINER ID` matches the hostname displayed in the previous step.

    Note: Outside of troubleshooting, you should never need to interact with the containers in a service directly.

1. Execute the curl command from the **other** Linux node in your cluster (the one where the container is not running). 

    ```bash
    $ curl localhost:8080

    Served by host: b738b7709d4b
    ```

    Notice that the service responds regardless of which Linux host you execute the `curl` command from. This is the ingress routing mesh at work. Port 8080 is published across the cluster, and any requests that come in will be routed to a node where a task is running. In our case, we only have one replica, so we only have one container that can respond. 

    > Note: At the time of writing this lab Windows hosts could not participate in the ingress routing mesh. However, with the latest update to Windows (1709), this restriction no longer exists.

1. List the networks in your swarm cluster

    ```bash
    $ docker network ls

    NETWORK ID          NAME                DRIVER              SCOPE
    36cbd99c755e        bridge              bridge              local
    ca57db11455b        docker_gwbridge     bridge              local
    a9d94a365900        host                host                local
    mbvc90x4n2i1        ingress             overlay             swarm1
    a9562964665a        none                null                local
    ```

    Notice that the `ingress` overlay network that is scoped to `swarm`. This is the network that handles all incoming requests to the cluster. All nodes in the cluster are automatically joined to the `ingress` network when they are created.

1. With Docker Swarm you can easily increase the number of tasks in a service. This can be done to add horizontal scalability or provide fault tolerance (or both).

    Scale the number of tasks running in the service. 

    **Make sure you are back in the console for `node1`**

    ```bash
    $ docker service scale \
    --detach=true \
    hostname=5

    hostname scaled to 5
    ```

1. List the tasks now running in the service

    ```bash
    $ docker service ps hostname

    NAME                IMAGE                                 NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
    nrbwg96sit7x        hostname.1          dockersamples/linux-hostname:latest   node2               Running             Running 2 minutes ago
    xzcyl9n3u47a        hostname.2          dockersamples/linux-hostname:latest   node2               Running             Running 2 minutes ago
    6nkyz2rsi44y        hostname.3          dockersamples/linux-hostname:latest   node1               Running             Running 2 minutes ago
    n65wj1o4luc2        hostname.4          dockersamples/linux-hostname:latest   node2               Running             Running 2 minutes ago
    4xw1kl3a9t24        hostname.5          dockersamples/linux-hostname:latest   node1               Running             Running 2 minutes ago
    ```

    Notice the tasks have been spread  between `node` and `node2`

1. Another feature of swarm is load balancing. Incoming requests will balanced between the nodes in the cluster using IPVS. Issue multple curl commands to the service to see this in action:

    ```bash
    $ for ((i=1; i<=5; i+=1)); do curl localhost:8080; done;

    Served by host: 5e4c0354adf2
    Served by host: 90207eb2beaf
    Served by host: aacbc0791536
    Served by host: a85e7065bbda
    Served by host: fd1e67aaf7dd
    ```

    Notice that each request is serviced by a different container running in the cluster. Feel free to list the running containers on each of the two Linux nodes to verify that the host name displayed above match the containers IDs of the running tasks.

1. To avoid a port conflict later in the lab, remove the `hostanme` service

    ```bash
    $ docker service rm hostname

    hostname
    ```

## Deploying a Multi-OS Application with Docker Swarm

Up until this point we have only deployed a single service application, and that application was only running on Linux. In this next section we'll deploy a two service application, with one service (a Java-based web front end) running on Linux and the other service (a Microsoft SQL Server database) running on Windows.

1. Make sure you're on  `node1`

1. We need to create an overlay network enable communication between the services.

    ```bash
    $ docker network create -d overlay atsea

    foqztzic1x95kiuq9cuqwuldi
    ```

1. Deploy the database service

    ```bash
    $ docker service create \
    --name database \
    --endpoint-mode dnsrr \
    --network atsea \
    --publish mode=host,target=1433 \
    --detach=true \
    sixeyed/atsea-db:mssql

    ywlkfxw2oim67fuf9tue7ndyi
    ```
    The service is created with the following parameters:

    * `--name`: Gives the service an easily remembered name
    * `--endpoint-mode`: Today all services running on Windows need to be started in DNS round robin mode.
    * `--network`: Attaches the containers from our service to the `atsea` network
    * `--publish`: Exposes port 1433 but only on the Windows host. 
    * `--detach`: Runs the service in the background
    * Our service is based off the image `sixeyed/atsea-db:mssql`

    > Note: Even though this is a Windows-based service we still issued the command from our Linux manager. As noted before all commands against the Swarm need to be issued from a manager regardless of the underlying OS of the service.
    >
    > Note: We didn't have to tell Swarm explicitly to schedule the service on to a Windows worker, that was automatically handled by our manager node. 

1. Check the status of your service

    ```bash
    $ docker service ps database

    ID                  NAME                IMAGE                          NODE                DESIRED STATE       CURRENT STATE      ERROR               PORTS
    rgwtocu21j0f        database.1          sixeyed/atsea-db:mssql   win00003R           Running             Running 3 minutesago
    ```

    > Note: Keep checking the status of the service until the `CURRENT STATE` is running. This usually takes 2-3 minutes

1. Start the web front-end service

    ```bash
    $ docker service create \
    --publish 8080:8080 \
    --network atsea \
    --name appserver \
    --detach=true \
    mikegcoleman/atsea_appserver:1.0

    tqvr2cxk31tr0ryel5ey4zmwr
    ```

1. List all the services running on your host

    ```bash
    $ docker service ls

    ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
    tqvr2cxk31tr        appserver           replicated          1/1                 mikegcoleman/atsea-appserver:1.0   *:8080->8080/
    tcp
    xkm68h7z3wsu        database            replicated          1/1                 sixeyed/atsea-db:mssql
    ```

1. Make sure both services are up and running (check the `Current State`)

    ```bash
    $ docker service ps $(docker service ls -q)

    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE
                ERROR               PORTS
    jhetafd6jd7u        database.1          sixeyed/atsea-db:mssql              win00003R           Running             Running 3 min
    utes ago                        *:64024->1433/tcp
    2cah7mw5a5c7        appserver.1         mikegcoleman/atsea-appserver:1.0   node1               Running             Running 6 min
    utes ago
    ```

1. Visit the running website by clicking the `8080` at the top of the PWD screen. The images being displayed on the web site are actually pulled from the SQL Server database. If they are displayed correctly then your app is functioning as expected. 

> Note: Our application code knows nothing about our networking code. The only thing it knows is that the database hostname is going to be `database`. So in our application code database connection string looks like this:
>
>   ```bash
>   jdbc:sqlserver://database;user=MyUserName;password=*****;
>   ```
>
> So long as the database service is started with the name `database` and is on the same Swarm network, the two services can talk. This is service discovery at work in Docker swam

## Using Docker Compose

In the previous steps we created our network and services manually. In reality we want everything to be as automated and reproducible as possible. Manual steps, of course, increase the possiblity of errors. 

To allow complex applications stacks to be rolled out easily Docker features compose files. Compose files are simply yaml files that provide a blueprint for an application including its services, networks, secrets, and just about any other Docker primitive it may need. 

In this section we'll redeploy our application using a Docker compose file.

Below is our Docker compose file. You'll notice the syntax is easy to interpret since it matches command line parameters pretty closely. 

```yaml
version: "3.2"

services:

database:
    image: sixeyed/atsea-db:mssql
    ports:
    - mode: host
        target: 1433
    networks:
    - atsea
    deploy:
    endpoint_mode: dnsrr

appserver:
    image: mikegcoleman/atsea_appserver:1.0
    ports:
    - target: 8080
        published: 8080
    networks:
    - atsea

networks:
    atsea:
```

Before we can deploy via the compose file, we'll need to remove our existing application. 

1. Remove the existing services

    ```bash
    $ docker service rm appserver

    appserver

    $ docker service rm database

    database
    ```

1. Remove the existing network

    ```bash
    $ docker network rm atsea

    atsea
    ```

    Swarm offers us the `stack` primitive to represent an application. A `stack` is a set of resources that are deployed together to represent an application. In the following steps we'll deploy our AtSea stack using the compose file. 

1. Clone the workshop repo onto `node1`

    ```bash
    $ git clone https://github.com/mikegcoleman/hybrid-swarm.git

    Cloning into 'hybrid-swarm'...
    remote: Counting objects: 54, done.
    remote: Compressing objects: 100% (25/25), done.
    remote: Total 54 (delta 16), reused 54 (delta 16), pack-reused 0
    Unpacking objects: 100% (54/54), done.
    ```

1. Change into the `hybrid-swarm` directory

    ```bash
    $ cd hybrid-swarm
    ```

1. Deploy the application stack using the compose file

    ```bash
    $ docker stack deploy -c docker-compose.yaml atsea

    Creating network atsea_atsea
    Creating service atsea_database
    Creating service atsea_appserver
    ```

    * `-c` specified which file should be used to deploy the stack
    * `atsea` is the name of our stack

    > Notice that even though our network was listed last in our compose file, it was created first. Swarm knows that the two services need to connect to the network, so it's created first to allow this to happen

1. List the services

    ```bash
    $ docker service ls

    ID                  NAME                MODE                REPLICAS            IMAGE                              PORTS
    miy9k6siatm1        atsea_appserver     replicated          1/1                 mikegcoleman/atsea_appserver:1.0   *:8080->8080/tcp
    8tqlpvq8ewf0        atsea_database      replicated          1/1                 sixeyed/atsea-db:mssql
    ```

1. Check to make sure the services are up and running.

    ```bash
    $ docker service ps $(docker service ls -q)

    ID                  NAME                IMAGE                              NODE                DESIRED STATE       CURRENT STATE               ERROR               PORTS
    8tw90grd8kli        atsea_appserver.1   mikegcoleman/atsea_appserver:1.0   node1               Running             Running abouta minute ago
    wh77n4p6osx5        atsea_database.1    sixeyed/atsea-db:mssql             win0001BD           Running             Running abouta minute ago                       *:30165->1433/tcp
    ```

    When the services have a current state of `running` move on to the next step.

1. Click the `8080` at the top of the screen to verify your application is running.

## Upgrades and Rollback

A common scenario is the need to upgrade an application or application component. In this section we are going to unsuccessfully attempt to ugrade the web front-end. We'll rollback from that attempt, and then perform a successful upgrade.

1. Make sure you're on `node1`

    To upgrade our application we're simply going to roll out an updated Docker image. In this case version `2.0`.

1. Upgrade the Appserver service to version 2.0

    ```bash
    $ docker service update \
    --image mikegcoleman/atsea_appserver:2.0 \
    --update-failure-action pause \
    --detach=true \
    atsea_appserver

    atsea_appserver
    ```

    > Notice that our service name is prepended with the name of our stack `atsea_appserver`

1. Check on the status of the upgrade

    ```bash
    $ docker service ps atsea_appserver

    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE
                        ERROR                              PORTS
    pjt4g23r0oo1        appserver.1         mikegcoleman/atsea-appserver:2.0   node1               Running             Starting less
    than a second ago
    usx1sk2gtoib         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node2               Shutdown            Failed 5 seco
    nds ago              "task: non-zero exit (143): do…"
    suee368vg3r1         \_ appserver.1     mikegcoleman/atsea-appserver:1.0   node1               Shutdown            Shutdown 24 seconds ago
    ```

    Clearly there is some issue, as the containers are failing to start. 

1. Check on the status of the update

    ```bash
    $ docker service inspect -f '{{json .UpdateStatus}}' atsea_appserver | jq

    {
      "State": "paused",
      "StartedAt": "2017-10-14T00:38:30.188743311Z",
      "Message": "update paused due to failure or early termination of task umidyotoa5i4gryk5vsrutwrq"
    }
    ```

    Because we had set ` --update-failure-action` to pause, Swarm paused the update. 

    In the case of failed upgrade, Swarm makes it easy to recover. Simply issue the `--rollback` command to the service. 

1. Roll the service back to the original version

    ```bash
    $ docker service update \
    --rollback \
    --detach=true \
    atsea_appserver

    atsea_appserver
    ```

1. Check the status of the update
    ```bash
    $ docker service inspect -f '{{json .UpdateStatus}}' atsea_appserver | jq

    {
    "State": "rollback_started",
    "StartedAt": "2017-10-26T08:51:03.644872494Z",
    "Message": "manually requested rollback"
    }
    ```

1. Check on the status of the service

    ```bash
    $ docker service ps atsea_appserver

    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE                 ERROR      PORTS
    yoswxm44q9vg        appserver.1         mikegcoleman/atsea_appserver:1.0    node2               Running             Running 11 seconds ago
    lacfi5xiu6e7         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Shutdown 25 seconds ago
    tvcr9dwvm578         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Failed 49 seconds ago         "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Failed about a minute ago     "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1               Shutdown            Shutdown about a minute ago
    ```

    The top line shows the service is back on the `1.0` version, and running. 

1. Visit the website to makes sure it's running

    That was a simulated upgrade failure and rollback. Next the service will be successfully upgraded to version 3 of the app. 

1. Upgrade to version 3

    ```bash
    $ docker service update \
    --image mikegcoleman/atsea_appserver:3.0 \
    --update-failure-action pause \
    --detach=true \
    atsea_appserver

    atsea_appserver
    ```

1. Check the status of the upgrade

    ```bash
    $ docker service ps atsea_appserver

    ID                  NAME                IMAGE                               NODEDESIRED STATE       CURRENT STATE             ERROR                              PORTS
    ytygwmyhumrt        appserver.1         mikegcoleman/atsea-appserver:3.0   node1Running             Running 29 seconds ago
    zjkmbjw7u8e0         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1Shutdown            Shutdown 47 seconds ago
    wemedok12frl         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1Shutdown            Failed 2 minutes ago      "task: non-zero exit (143): do…"
    u6wd7wje82zn         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1Shutdown            Failed 2 minutes ago      "task: non-zero exit (143): do…"
    ```

1. Once the status reports back "Running xx seconds", reload website the website once again to verify that the new version has been deployed

## Scale the front end

The new update has really increased traffic to the site. As a result we need to scale our web front end out. This is done by issuing a `docker service update` and specifying the number of replicas to deploy.

1. Scale to 6 replicas of the web front-end

    ```bash
    $  docker service update \
    --replicas=6 \
    --detach=true \
    atsea_appserver

    atsea_appserver
    ```

1. Check the status of the update

    ```bash
    $ docker service ps atsea_appserver

    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE             ERROR
      PORTS
    vfbzj3axoays        appserver.1         mikegcoleman/atsea-appserver:3.0   node1               Running             Running 2 minutes ago

    yoswxm44q9vg         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node2               Shutdown            Shutdown 2 minutes ago
    tvcr9dwvm578         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Failed 5 minutes ago      "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Failed 6 minutes ago      "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1               Shutdown            Shutdown 7 minutes ago
    i474a8emgwbc        appserver.2         mikegcoleman/atsea-appserver:3.0   node2               Running             Starting 30 seconds ago
    gu7rphvp2q3l        appserver.3         mikegcoleman/atsea-appserver:3.0   node2               Running             Starting 30 seconds ago
    gzjdye1kne33        appserver.4         mikegcoleman/atsea-appserver:3.0   node1               Running             Running 7 seconds ago
    u596cqkgf2aa        appserver.5         mikegcoleman/atsea-appserver:3.0   node2               Running             Starting 30 seconds ago
    jqkokd2uoki6        appserver.6         mikegcoleman/atsea-appserver:3.0   node1               Running             Running 12 seconds ag
    ```

Docker is starting up 5 new instances of the appserver, and is placing them across both the nodes in the cluster.

When all 6 nodes are running, move on to the next step.

## Failure and recovery
The next exercise simulates a node failure. When a node fails the containers that were running there are, of course, lost as well. Swarm is constantly monitoring the state of the cluster, and when it detects an anomaly it attemps to bring the cluster back in to compliance.

In it's current state, Swarm expects there to be six instances of the appserver. When the node "fails" three of those instances will go out of service. 

1. Putting a node into *drain* mode forces it to stop all the running containers it hosts, as well as preventing it from running any additional containers. 

    ```bash
    $ docker node update \
    --availability=drain \
    node2

    node2
    ```

1. Check the status of the service

    ```bash
    $ docker service ps atsea_appserver

    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE             ERROR  PORTS
    vfbzj3axoays        appserver.1         mikegcoleman/atsea-appserver:3.0   node1               Running             Running 8 minutes ago
    yoswxm44q9vg         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node2               Shutdown            Shutdown 8 minutes ago
    tvcr9dwvm578         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Failed 11 minutes ago     "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     mikegcoleman/atsea-appserver:2.0   node1               Shutdown            Failed 12 minutes ago     "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1               Shutdown            Shutdown 12 minutes ago
    zmp7mfpme2go        appserver.2         mikegcoleman/atsea-appserver:3.0   node1               Running             Starting 5 seconds ago
    i474a8emgwbc         \_ appserver.2     mikegcoleman/atsea-appserver:3.0   node2               Shutdown            Shutdown 5 seconds ago
    l7gxju3x6zx8        appserver.3         mikegcoleman/atsea-appserver:3.0   node1               Running             Starting 5 seconds ago
    gu7rphvp2q3l         \_ appserver.3     mikegcoleman/atsea-appserver:3.0   node2               Shutdown            Shutdown 5 seconds ago
    gzjdye1kne33        appserver.4         mikegcoleman/atsea-appserver:3.0   node1               Running             Running 5 minutes ago
    ure9u7li7myv        appserver.5         mikegcoleman/atsea-appserver:3.0   node1               Running             Starting 5 seconds ago
    u596cqkgf2aa         \_ appserver.5     mikegcoleman/atsea-appserver:3.0   node2               Shutdown            Shutdown 5 seconds ago
    jqkokd2uoki6        appserver.6         mikegcoleman/atsea-appserver:3.0   node1               Running             Running 6 minutes ago
    ```

    The output above shows the containers that were running on `node2` have been shut down and are being restarted on `node1`

1. List the status of our services

    ```bash
    $ docker service ls

    ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
    qbeqlc6v0g0z        appserver           replicated          6/6                 mikegcoleman/atsea-appserver:3.0   *:8080->8080/tcps3luy288gn9l        
    database            replicated          1/1                 sixeyed/atsea-db:mssql
    ```

    In a minute or two all the services should be restarted, and Swarm will report back that it has 6 of the expected 6 containers running
