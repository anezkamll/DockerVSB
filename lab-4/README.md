# Docker - Linux (Part 4): Docker Swarm

So far all of the previous exercises have been based around running a single container on a single host.

This section will cover how to use multiple hosts to provide fault tolerance as well as increased performance. As part of that discussion it will also provide an overview of Docker's multi-host networking capabilities.


> * [Task 1: Build your own cluster](#Task1)
> * [Task 2: Overlay Networking](#Task2)
> * [Task 3: Deploying an Application with Docker Swarm](#Task3)
> * [Task 4: Upgrades and Rollback](#Task4)
> * [Task 5: Scale the front end](#Task5)
> * [Task 6: Failure and recovery](#Task6)


## <a name="Task1"></a>Task 1: Build your own cluster
The next part of the lab will start with the deployment of a 3 node Docker swarm cluster.

### Build your cluster

Note: If you have just completed a previous part of the workshop, please close that session and start a new one.

1. In the PWD interface click `+ Add new instance` to instantiate a linux node

  > Note: If you've closed the PWD interface, simply go back to [the PWD site](https://hybrid.play-with-docker.com/)

2. Repeat step 1 to add a second node to the cluster.

3. Click the  `+ Add New Instance`

  There are now three standalone Docker hosts.

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

7. Switch to the third node and paste the same command at the Powershell prompt

    ```
    $ docker swarm join --token SWMTKN-1-53ao1ihu684vpcpjuf332bi4et27qiwxajefyqryyj4o0indm7-b2zc5ldudkxcmf6kcncft8t12 192.168.0
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

    Commands against the swarm can only be issued from the manager node. Attempting to run the above command against `node2` or `node3` would result in an error.

    ```
    $ docker node ls
    Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
    ```

## <a name="Task2"></a>Task 2: Overlay Networking

### Overlay Networking Overview

Overlay networks in Docker are software defined networks that span multiple hosts (unlike a bridge network which is limited to a single Docker host). This allows containers on different hosts to easily communicate on the Docker networking fabric (vs having to move out to the host's network).

This next section covers building an overlay network and having two containers communicate with each other.

1. Remove the existing Alpine containers

        ```
        $ docker container rm --force $(docker ps --all --quiet)
        e65629beeb57
        5cc5eeaf703b
        ```

2. Create a new overlay network (`-d` specifies the networking driver to use, if it's omitted `bridge` is the default).

        ```
        $ docker network create --attachable -d overlay myoverlay
        z16nhzxwbeukjnz3e6nk2159p
        ```

        > Note: We have to use the `--attachable` flag because by default you cannot use `docker run` on overlay networks that are part of a swarm. The preferred method is to use a Docker *service* which is covered later in the workshop.

3. List the networks on the host to verify that the `myoverlay` network was created.

        ```
        $ docker network ls
        NETWORK ID          NAME                DRIVER              SCOPE
        edf9dc771fc4        bridge              bridge              local
        e5702f60b7c9        docker_gwbridge     bridge              local
        7d6b733ee498        host                host                local
        rnyatjul3qhn        ingress             overlay             swarm
        52fb9de4ad1c        mybridge            bridge              local
        z16nhzxwbeuk        myoverlay           overlay             swarm
        dbd52ffda3ae        none                null                local
        ```

4. Create an Alpine container and attach it to the `myoverlay` network.

        ```
        $ docker container run \
          --detach \
          --network myoverlay \
          --name alpine_host \
          alpine top
        a604aa48660835aeec75f3239964d35c334bcdf33d1b5574c319aaf344c2119a
        ```

5. Move to `node2`

6. List the available networks

        ```
        $ docker network ls
        NETWORK ID          NAME                DRIVER              SCOPE
        3bc2a78be20f        bridge              bridge              local
        641bdc72dc8b        docker_gwbridge     bridge              local
        a5ef170a2758        host                host                local
        rnyatjul3qhn        ingress             overlay             swarm
        3dec80db87e4        none                null                local
        ```

Notice anything out of the ordinary? Where's the `myoverlay` network?

Docker won't extend the network to hosts where it's not needed. In this case, there are no containers attached to `myoverlay` on `node2` so the network has not been extended to the host.

7. Start an alpine container and attach it to `myoverlay`

        ```
        $ docker container run \
          --detach \
          --network myoverlay \
          --name alpine_client \
          alpine top
        Unable to find image 'alpine:latest' locally
        latest: Pulling from library/alpine
        88286f41530e: Pull complete
        Digest: sha256:f006ecbb824d87947d0b51ab8488634bf69fe4094959d935c0c103f4820a417d
        Status: Downloaded newer image for alpine:latest
        5d67e360d8e42c618dc8ea40ecd745280a8002652c7bcdc7982cb5c6cdd4fd13
        ```

8. List the available networks on `node2`

        ```
        $ docker network ls
        NETWORK ID          NAME                DRIVER              SCOPE
        3bc2a78be20f        bridge              bridge              local
        641bdc72dc8b        docker_gwbridge     bridge              local
        a5ef170a2758        host                host                local
        rnyatjul3qhn        ingress             overlay             swarm
        z2fh5l7g1b4k        myoverlay           overlay             swarm
        3dec80db87e4        none                null                local
        ```

The `myoverlay` network is now available on `node2`

9. Ping `apine_host`

        ```
        $ docker exec alpine_client ping -c 5 alpine_host
        PING alpine_host (10.0.0.2): 56 data bytes
        64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.244 ms
        64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.122 ms
        64 bytes from 10.0.0.2: seq=2 ttl=64 time=0.166 ms
        64 bytes from 10.0.0.2: seq=3 ttl=64 time=0.201 ms
        64 bytes from 10.0.0.2: seq=4 ttl=64 time=0.137 ms
        ```
Networking also works betwen Linux and Windows nodes

10. Move to the `node3`

11. Ping `alpine_host` from the `node3`

        ```
        docker container run \
          --rm \
          --network myoverlay \
          alpine ping alpine_host

        Pinging alpine_host [10.0.0.2] with 32 bytes of data:
        Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
        Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
        Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
        Reply from 10.0.0.2: bytes=32 time<1ms TTL=64
        ```

> Note: In some cases it may take a few seconds for the client to find the alpine host resutling in PING timeouts. If this happens, simply retry the above command.

## <a name="Task3"></a>Task 3: Deploying an Application with Docker Swarm
This lab will deploy a two service application.  The application features a Java-based web front end running on Linux, and a Microsoft SQL server running on Windows.

### Deploying an Application with Docker Swarm

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
      --publish mode=host,target=6379 \
      --detach=true \
    redis
    ywlkfxw2oim67fuf9tue7ndyi
    ```
    The service is created with the following parameters:

    * `--name`: Gives the service an easily remembered name
    * `--endpoint-mode`: Today all services running on Windows need to be started in DNS round robin mode.
    * `--network`: Attaches the containers from our service to the `atsea` network
    * `--publish`: Exposes port 6379 but only on the host.
    * `--detach`: Runs the service in the background
    * Our service is based off the image `sixeyed/atsea-db:mssql`

4. Check the status of your service

    ```
    $ docker service ps database
    ID                  NAME                IMAGE                          NODE                DESIRED STATE       CURRENT STATE      ERROR               PORTS
    rgwtocu21j0f        database.1          redis:latest                   node3           Running             Running 3 minutes ago
    ```

    > Note: Keep checking the status of the service until the `CURRENT STATE` is running. This usually takes 2-3 minutes

5. Start the web front-end service

    ```
    $ docker service create \
    --publish 8000:8000 \
    --network atsea \
    --name appserver \
    --detach=true \
    solarwinds/awesomeapp:1.0
    tqvr2cxk31tr0ryel5ey4zmwr
    ```

6. List all the services running on your host

    ```
    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
    tqvr2cxk31tr        appserver           replicated          1/1                 solarwinds/awesomeapp:1.0           *:8000->8000/
    tcp
    xkm68h7z3wsu        database            replicated          1/1                 redis:latest
    ```

7. Make sure both services are up and running (check the `Current State`)

    ```
    $ docker service ps $(docker service ls -q)
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE
                ERROR               PORTS
    jhetafd6jd7u        database.1          redis:latest                node2              Running             Running 3 min
    utes ago                        *:64024->1433/tcp
    2cah7mw5a5c7        appserver.1         solarwinds/awesomeapp:1.0                         node1               Running             Running 6 min
    utes ago
    ```

8. Visit the running website by clicking the `8000` at the top of the PWD screen.

We've successfully deployed our application.  

Another key point is that our application code knows nothing about our networking code. The only thing it knows is that the database hostname is going to be `database`. So in our application code database connection string looks like this;


So long as the database service is started with the name `database` and is on the same Swarm network, the two services can talk.


## <a name="Task4"></a>Task 4: Upgrades and Rollback
A common scenario is the need to upgrade an application or application component. In this section we are going to unsuccessfully attempt to ugrade the web front-end. We'll rollback from that attempt, and then perform a successful upgrade.

### Upgrades and Rollback

1. Make sure you're on `node1`

    To upgrade our application we're simply going to roll out an updated Docker image. In this case version `2.0`.

2. Upgrade the Appserver service to version 2.0

    ```
    $ docker service update \
    --image solarwinds/awesomeapp:2.0 \
    --update-failure-action pause \
    --detach=true \
    appserver
    ```

3. Check on the status of the upgrade

    ```
    $ docker service ps appserver
    ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE
                        ERROR                              PORTS
    pjt4g23r0oo1        appserver.1         solarwinds/awesomeapp:2.0   node1               Running             Starting less
    than a second ago
    usx1sk2gtoib         \_ appserver.1     solarwinds/awesomeapp:2.0   node2               Shutdown            Failed 5 seco
    nds ago              "task: non-zero exit (143): do…"
    suee368vg3r1         \_ appserver.1     solarwinds/awesomeapp:1.0   node1               Shutdown            Shutdown 24 seconds ago
    ```

    Clearly there is some issue, as the containers are failing to start.

4. Check on the status of the update

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
    yoswxm44q9vg        appserver.1         solarwinds/awesomeapp:1.0    node2               Running             Running 11 seconds ago
    lacfi5xiu6e7         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Shutdown 25 seconds ago
    tvcr9dwvm578         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Failed 49 seconds ago         "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Failed about a minute ago     "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     solarwinds/awesomeapp:1.0    node1               Shutdown            Shutdown about a minute ago
    ```

    The top line shows the service is back on the `1.0` version, and running.

7. Visit the website to makes sure it's running

    That was a simulated upgrade failure and rollback. Next the service will be successfully upgraded to version 3 of the app.

8. Upgrade to version 3

    ```
    $ docker service update \
    --image solarwinds/awesomeapp:3.0 \
    --update-failure-action pause \
    --detach=true \
    appserver
    appserver
    ```

9. Check the status of the upgrade

    ```
    $ docker service ps appserver
    ID                  NAME                IMAGE                               NODEDESIRED STATE       CURRENT STATE             ERROR                              PORTS
    ytygwmyhumrt        appserver.1         solarwinds/awesomeapp:3.0   node1Running             Running 29 seconds ago
    zjkmbjw7u8e0         \_ appserver.1     solarwinds/awesomeapp:1.0    node1Shutdown            Shutdown 47 seconds ago
    wemedok12frl         \_ appserver.1     solarwinds/awesomeapp:2.0   node1Shutdown            Failed 2 minutes ago      "task: non-zero exit (143): do…"
    u6wd7wje82zn         \_ appserver.1     solarwinds/awesomeapp:2.0   node1Shutdown            Failed 2 minutes ago      "task: non-zero exit (143): do…"
    ```

10. Once the status reports back "Running xx seconds", reload website the website once again to verify that the new version has been deployed

## <a name="Task5"></a>Task 5: Scale the front end
Sometimes you need to scale your application for peaks like "Black Friday sales". With Docker Swarm is quite easy.

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
    vfbzj3axoays        appserver.1         solarwinds/awesomeapp:3.0   node1               Running             Running 2 minutes ago

    yoswxm44q9vg         \_ appserver.1     solarwinds/awesomeapp:1.0    node2               Shutdown            Shutdown 2 minutes ago
    tvcr9dwvm578         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Failed 5 minutes ago      "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Failed 6 minutes ago      "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     solarwinds/awesomeapp:1.0    node1               Shutdown            Shutdown 7 minutes ago
    i474a8emgwbc        appserver.2         solarwinds/awesomeapp:3.0   node2               Running             Starting 30 seconds ago
    gu7rphvp2q3l        appserver.3         solarwinds/awesomeapp:3.0   node2               Running             Starting 30 seconds ago
    gzjdye1kne33        appserver.4         solarwinds/awesomeapp:3.0   node1               Running             Running 7 seconds ago
    u596cqkgf2aa        appserver.5         solarwinds/awesomeapp:3.0   node2               Running             Starting 30 seconds ago
    jqkokd2uoki6        appserver.6         solarwinds/awesomeapp:3.0   node1               Running             Running 12 seconds ag
    ```

Docker is starting up 5 new instances of the appserver, and is placing them across both the nodes in the cluster.

When all 6 nodes are running, move on to the next step.

## <a name="Task6"></a>Task 6: Failure and recovery
The next exercise simulates a node failure. When a node fails the containers that were running there are, of course, lost as well. Swarm is constantly monitoring the state of the cluster, and when it detects an anomoly it attemps to bring the cluster back in to compliance.

### Failure and recovery


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
    vfbzj3axoays        appserver.1         solarwinds/awesomeapp:3.0   node1               Running             Running 8 minutes ago
    yoswxm44q9vg         \_ appserver.1     solarwinds/awesomeapp:1.0    node2               Shutdown            Shutdown 8 minutes ago
    tvcr9dwvm578         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Failed 11 minutes ago     "task: non-zero exit (143): do…"
    xg4274jpochx         \_ appserver.1     solarwinds/awesomeapp:2.0   node1               Shutdown            Failed 12 minutes ago     "task: non-zero exit (143): do…"
    z7toh7jwk8qf         \_ appserver.1     solarwinds/awesomeapp:1.0    node1               Shutdown            Shutdown 12 minutes ago
    zmp7mfpme2go        appserver.2         solarwinds/awesomeapp:3.0   node1               Running             Starting 5 seconds ago
    i474a8emgwbc         \_ appserver.2     solarwinds/awesomeapp:3.0   node2               Shutdown            Shutdown 5 seconds ago
    l7gxju3x6zx8        appserver.3         solarwinds/awesomeapp:3.0   node1               Running             Starting 5 seconds ago
    gu7rphvp2q3l         \_ appserver.3     solarwinds/awesomeapp:3.0   node2               Shutdown            Shutdown 5 seconds ago
    gzjdye1kne33        appserver.4         solarwinds/awesomeapp:3.0   node1               Running             Running 5 minutes ago
    ure9u7li7myv        appserver.5         solarwinds/awesomeapp:3.0   node1               Running             Starting 5 seconds ago
    u596cqkgf2aa         \_ appserver.5     solarwinds/awesomeapp:3.0   node2               Shutdown            Shutdown 5 seconds ago
    jqkokd2uoki6        appserver.6         solarwinds/awesomeapp:3.0   node1               Running             Running 6 minutes ago
    ```

    The output above shows the containers that werer running on `node2` have been shut down and are being restarted on `node`

3. List the status of our services

    ```
    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
    qbeqlc6v0g0z        appserver           replicated          6/6                 solarwinds/awesomeapp:3.0   *:8000->8000/tcps3luy288gn9l        
    database            replicated          1/1                 redis:latest
    ```

    In a minute or two all the services should be restarted, and Swarm will report back that it has 6 of the expected 6 containers running
