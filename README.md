# Deploying Hybrid Applications with Docker Swarm



#### What is "Orchestration"

If you heard about containers you've probably heard about orchstration as well. Conatiner orchestrators, such as Docker Swarm and Kubernetes, provide a ton of functionality around managing and deploying containerized applications. But the two most fundamental things they do are clustering and scheduling (that's not all they do by a long shot, but they are arguably the two most important fuctions).

Clustering is the concept of taking a group of machines and treating them as a single computing resource. These machines are capable of accepting any workload becaues they all offer the same capabilities. These clustered machines don't have to be running on the same infrastructure - they could be a  mix of bare metal and VMs for instance.  

Scheduling is the process of deciding where a workload should reside. When an admin starts a new instance of website she can decide what region it needs to go on or if it should be on bare metal or in the cloud. The scheduler will make that happen. Schedulers also make sure that the application maintains its desired state. For example, if there were 10 copies of a web site running, and one of them crashed, the scheduler would know this and start up a new instance to take the failed one's place. 


With Docker ther is a built-in orchestrator: Docker Swarm. It provides both clustering and scheduling as well as many other advanced services. 

The next part of the lab will start with the deployment of a 3 node Docker swarm cluster. 

### Build your cluster

Note: If you have just completed a previous part of the workshop, please close that session and start a new one.

1. In your broswer navigate to [Play with Docker](https://dockercon.play-with-docker.com)

1. In the PWD interface click `+ Add new instance` to instantiate a linux node

2. Repeat step 1 to add a second node to the cluster.

3. Click the `Windows containers` slider and then click `+ Add New Instance`

    There are now three standalone Docker hosts: 2 linux and 1 Windows. 

4. In the console for `node1` initialize Docker Swarm

    ```
    $ docker swarm init --advertise-addr eth0

    Swarm initialized: current node (ujsz5fd1ozr410x1ywuix7sik) is now a manager.

    To add a worker to this swarm, run the following command:

        docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
    ```

    `node1` is now a Swarm manager node. Manager nodes are responsible for ensuring the integrity of the cluster as well as managing running services. 

5. Copy the `docker swarm join` output from `node1`

6. Switch to `node2` and paste in the command. **Be sure to copy the output from your PWD window, not from this lab guide**

    ```
    $ docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0.13:2377

    This node joined a swarm as a worker.
    ```

7. Switch to the Windows node and paste the same command at the Powershell prompt

    ```
    PS C:\> docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0
    .13:2377

    This node joined a swarm as a worker.
    ```

    The three nodes have now been clustered into a single Docker swarm. An important thing to note is that clusters can be made up of Linux nodes, Windows nodes, or a combination of both. 

8. Switch back to `node1`

9. List the nodes in the cluster

    ```
    $ docker node ls
    
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    uqqgsvc1ykkgoamaatcun89wt *   node1               Ready               Active              Leader
    g4demzyr7sd2ngpa8wntbo0l4     node2               Ready               Active
    xflngp99u1r9pn7bryqbbrrvq     win000046           Ready               Active
    ```

10. Commands against the swarm can only be issued from the manager node. Attempting to run the above command from the `Windows` 

    ```
    PS C:\ docker node ls

    Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
    ```

11. Move back to `node1` and promote the Windows node to a manager role:

    > **Note** Be sure to use the name of your Windows node

    ```
    $ docker node promote winxxxxx

    Node winxxxxx promoted to a manager in the swarm.
    ```

12. Move back to the `Windows` node and run the `docker node ls` command again

    ```
    PS C:\ docker node ls

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    ofetnt63m2f6uthres6jnne08 *   win000198           Ready               Active              Reachable
    q8swkmaldrhm2abacbzafscbo     node1               Ready               Active              Leader
    ymj4gg6bnfvgc70j8r9zef593     node2               Ready               Active
    ```

13. Move back to the `node1` node and demote the Windows node back to a worker.

    ```
    $ docker node demote winxxxxx

    Manager winxxxx demoted in the swarm.
    ```


## Deploying Swarm services

While we have been using `docker run` to instantiate docker containers on our Swarm cluster, the preferred way to actually run applications is via a  [*service*](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/).

Services are an abstraction in Docker that represent an application or component of an application. For instance a web front end service connecting to a database backend service. You can deploy an application made up of a single service. In fact, this is quite common when [modernizing traditional applications](https://goto.docker.com/rs/929-FJL-178/images/SB_MTA_04.14.2017.pdf). 

The service construct provides a host of useful features including:

* Load balancing
* Layer 4 and layer 7 routing meshes
* Desired state reconciliation
* Service discovery
* Helthchecks
* Upgrades and rollback
* Scaling

This workshop cannot possibly cover all these topics, but will address several key points. 

This lab will deploy a two service application.  The application features a Java-based web front end running on Linux, and a Microsoft SQL server running on Windows. 

### Deploying a Multi-OS Application with Docker Swarm

1. Move to `node1`

2. Create an overlay network for the application

    ```
    $ docker network create -d overlay atsea
    
    foqztzic1x95kiuq9cuqwuldi
    ```

3. Deploy the database service

    ```
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

4. Check the status of your service

    ```
    $ docker service ps database
    
    ID                  NAME                IMAGE                          NODE                DESIRED STATE       CURRENT STATE      ERROR               PORTS
    rgwtocu21j0f        database.1          sixeyed/atsea-db:mssql   win00003R           Running             Running 3 minutesago
    ```

    > Note: Keep checking the status of the service until the `CURRENT STATE` is running. This usually takes 2-3 minutes

5. Start the web front-end service

    ```
    $ docker service create \
    --publish 8080:8080 \
    --network atsea \
    --name appserver \
    --detach=true \
    mikegcoleman/atsea_appserver:1.0
    
    tqvr2cxk31tr0ryel5ey4zmwr
    ```

6. List all the services running on your host

    ```
    $ docker service ls
    
    ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
    tqvr2cxk31tr        appserver           replicated          1/1                 dockersamples/atsea-appserver:1.0   *:8080->8080/
    tcp
    xkm68h7z3wsu        database            replicated          1/1                 sixeyed/atsea-db:mssql
    ```

7. Make sure both services are up and running (check the `Current State`)

    ```
    $ docker service ps $(docker service ls -q)
    
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE
                ERROR               PORTS
    jhetafd6jd7u        database.1          sixeyed/atsea-db:mssql              win00003R           Running             Running 3 min
    utes ago                        *:64024->1433/tcp
    2cah7mw5a5c7        appserver.1         dockersamples/atsea-appserver:1.0   node1               Running             Running 6 min
    utes ago
    ```

8. Visit the running website by clicking the `8080` at the top of the PWD screen.

We've successfully deployed our application. One thing to note is that we did not have to tell Swarm to put the database on the Windows node or the Java webserver on the Linux node. It was able to sort that out by itself. 

Another key point is that our application code knows nothing about our networking code. The only thing it knows is that the database hostname is going to be `database`. So in our application code database connection string looks like this;

```
jdbc:sqlserver://database;user=MyUserName;password=*****;
```

So long as the database service is started with the name `database` and is on the same Swarm network, the two services can talk. 

### Upgrades and Rollback

A common scenario is the need to upgrade an application or application component. In this section we are going to unsuccessfully attempt to ugrade the web front-end. We'll rollback from that attempt, and then perform a successful upgrade.

1. Make sure you're on `node1`

    To upgrade our application we're simply going to roll out an updated Docker image. In this case version `2.0`.

2. Upgrade the Appserver service to version 2.0

    ```
    $ docker service update \
    --image dockersamples/atsea-appserver:2.0 \
    --update-failure-action pause \
    --detach=true \
    appserver
    ```

3. Check on the status of the upgrade

    ```
    $ docker service ps appserver
    
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE
                        ERROR                              PORTS
    pjt4g23r0oo1        appserver.1         dockersamples/atsea-appserver:2.0   node1               Running             Starting less
    than a second ago
    usx1sk2gtoib         \_ appserver.1     dockersamples/atsea-appserver:2.0   node2               Shutdown            Failed 5 seco
    nds ago              "task: non-zero exit (143): do…"
    suee368vg3r1         \_ appserver.1     dockersamples/atsea-appserver:1.0   node1               Shutdown            Shutdown 24 seconds ago
    ```

    Clearly there is some issue, as the containers are failing to start. 

4. Check on the satus of the update

    ```
    $ docker service inspect -f '{{json .UpdateStatus}}' appserver | jq
    
    {
      "State": "paused",
      "StartedAt": "2017-10-14T00:38:30.188743311Z",
      "Message": "update paused due to failure or early termination of task umidyotoa5i4gryk5vsrutwrq"
    }
    ```

    Because we had set ` --update-failure-action` to pause, Swarm paused the update. 

    In the case of failed upgrade, Swarm makes it easy to recover. Simply issue the `--rollback` command to the service. 

5. Roll the service back to the original version

    ```
    $ docker service update \
    --rollback \
    --detach=true \
    appserver
    appserver
    ```

 6. Check on the status of the service

    ```
    $ docker service ps appserver
    
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE                 ERROR      PORTS
    yoswxm44q9vg        appserver.1         mikegcoleman/atsea_appserver:1.0    node2               Running             Running 11 seconds ago
    lacfi5xiu6e7         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Shutdown 25 seconds ago
    tvcr9dwvm578         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Failed 49 seconds ago         "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Failed about a minute ago     "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1               Shutdown            Shutdown about a minute ago
    ```

    The top line shows the service is back on the `1.0` version, and running. 

7. Visit the website to makes sure it's running

    That was a simulated upgrade failure and rollback. Next the service will be successfully upgraded to version 3 of the app. 

8. Upgrade to version 3

    ```
    $ docker service update \
    --image dockersamples/atsea-appserver:3.0 \
    --update-failure-action pause \
    --detach=true \
    appserver
    
    appserver
    ```

9. Check the status of the upgrade

    ```
    $ docker service ps appserver
    
    ID                  NAME                IMAGE                               NODEDESIRED STATE       CURRENT STATE             ERROR                              PORTS
    ytygwmyhumrt        appserver.1         dockersamples/atsea-appserver:3.0   node1Running             Running 29 seconds ago
    zjkmbjw7u8e0         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1Shutdown            Shutdown 47 seconds ago
    wemedok12frl         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1Shutdown            Failed 2 minutes ago      "task: non-zero exit (143): do…"
    u6wd7wje82zn         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1Shutdown            Failed 2 minutes ago      "task: non-zero exit (143): do…"
    ```

10. Once the status reports back "Running xx seconds", reload website the website once again to verify that the new version has been deployed

### Scale the front end

The new update has really increased traffic to the site. As a result we need to scale our web front end out. This is done by issuing a `docker service update` and specifying the number of replicas to deploy. 

1. Scale to 6 replicas of the web front-end

    ```
    $  docker service update \
    --replicas=6 \
    --detach=true \
    appserver
    
    appserver
    ```

2. Check the status of the update

    ```
    $ docker service ps appserver
    
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE             ERROR
      PORTS
    vfbzj3axoays        appserver.1         dockersamples/atsea-appserver:3.0   node1               Running             Running 2 minutes ago

    yoswxm44q9vg         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node2               Shutdown            Shutdown 2 minutes ago
    tvcr9dwvm578         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Failed 5 minutes ago      "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Failed 6 minutes ago      "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1               Shutdown            Shutdown 7 minutes ago
    i474a8emgwbc        appserver.2         dockersamples/atsea-appserver:3.0   node2               Running             Starting 30 seconds ago
    gu7rphvp2q3l        appserver.3         dockersamples/atsea-appserver:3.0   node2               Running             Starting 30 seconds ago
    gzjdye1kne33        appserver.4         dockersamples/atsea-appserver:3.0   node1               Running             Running 7 seconds ago
    u596cqkgf2aa        appserver.5         dockersamples/atsea-appserver:3.0   node2               Running             Starting 30 seconds ago
    jqkokd2uoki6        appserver.6         dockersamples/atsea-appserver:3.0   node1               Running             Running 12 seconds ag
    ```

Docker is starting up 5 new instances of the appserver, and is placing them across both the nodes in the cluster. 

When all 6 nodes are running, move on to the next step. 

### Failure and recovery
The next exercise simulates a node failure. When a node fails the containers that were running there are, of course, lost as well. Swarm is constantly monitoring the state of the cluster, and when it detects an anomoly it attemps to bring the cluster back in to compliance. 

In it's current state, Swarm expects there to be six instances of the appserver. When the node "fails" thre of those instances will go out of service. 

1. Putting a node into *drain* mode forces it to stop all the running containers it hosts, as well as preventing it from running any additional containers. 

    ```
    $ docker node update \
    --availability=drain \
    node2
    ```

2. Check the status of the service

    ```
    $ docker service ps appserver
    
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE             ERROR  PORTS
    vfbzj3axoays        appserver.1         dockersamples/atsea-appserver:3.0   node1               Running             Running 8 minutes ago
    yoswxm44q9vg         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node2               Shutdown            Shutdown 8 minutes ago
    tvcr9dwvm578         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Failed 11 minutes ago     "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     dockersamples/atsea-appserver:2.0   node1               Shutdown            Failed 12 minutes ago     "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     mikegcoleman/atsea_appserver:1.0    node1               Shutdown            Shutdown 12 minutes ago
    zmp7mfpme2go        appserver.2         dockersamples/atsea-appserver:3.0   node1               Running             Starting 5 seconds ago
    i474a8emgwbc         \_ appserver.2     dockersamples/atsea-appserver:3.0   node2               Shutdown            Shutdown 5 seconds ago
    l7gxju3x6zx8        appserver.3         dockersamples/atsea-appserver:3.0   node1               Running             Starting 5 seconds ago
    gu7rphvp2q3l         \_ appserver.3     dockersamples/atsea-appserver:3.0   node2               Shutdown            Shutdown 5 seconds ago
    gzjdye1kne33        appserver.4         dockersamples/atsea-appserver:3.0   node1               Running             Running 5 minutes ago
    ure9u7li7myv        appserver.5         dockersamples/atsea-appserver:3.0   node1               Running             Starting 5 seconds ago
    u596cqkgf2aa         \_ appserver.5     dockersamples/atsea-appserver:3.0   node2               Shutdown            Shutdown 5 seconds ago
    jqkokd2uoki6        appserver.6         dockersamples/atsea-appserver:3.0   node1               Running             Running 6 minutes ago
    ```

    The output above shows the containers that werer running on `node2` have been shut down and are being restarted on `node`

3. List the status of our services

    ```
    $ docker service ls
    
    ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
    qbeqlc6v0g0z        appserver           replicated          6/6                 dockersamples/atsea-appserver:3.0   *:8080->8080/tcps3luy288gn9l        
    database            replicated          1/1                 sixeyed/atsea-db:mssql
    ```

    In a minute or two all the services should be restarted, and Swarm will report back that it has 6 of the expected 6 containers running
