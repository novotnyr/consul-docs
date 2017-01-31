---
title: Consul 101
---


About Consul
===

[Consul](https://www.hashicorp.com/consul.html) is a distributed configuration key-value store and a scalable service discovery tool


Installing Consul
===

* Download Consul from the https://www.consul.io/downloads.html
* Extract Consul to a reasonable directory. The whole Consul is represented by a single binary, which might not be usual, but that's it.


Running Consul
===

Actually, we will be running a **Consul Agent** on our local machine. Let's start with the simplest solution: Consul agent in the development mode.


Running in the development mode
---

Consul Agent in the development mode will not use any persistence. In other words, it will be run in RAM.

```
./consul agent -dev
```

On some machines, you might get a complatint:

```
==> Error starting agent: Failed to get advertise address: Multiple private IPs found. Please configure one.
```

In this case, we have to specify explicit IP address to listent to:

```
./consul agent -dev-advertise=127.0.0.1
```

We will observe something along the lines of:

```
whitehall:Applications$ ./consul agent -dev -advertise=127.0.0.1
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
         Node name: 'whitehall'
        Datacenter: 'dc1'
            Server: true (bootstrap: false)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
             Atlas: <disabled>

==> Log data will now stream in as it occurs:

    2017/01/29 21:27:49 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state
    2017/01/29 21:27:49 [INFO] serf: EventMemberJoin: whitehall 127.0.0.1
    2017/01/29 21:27:49 [INFO] serf: EventMemberJoin: whitehall.dc1 127.0.0.1
    2017/01/29 21:27:49 [INFO] consul: adding LAN server whitehall (Addr: 127.0.0.1:8300) (DC: dc1)
    2017/01/29 21:27:49 [INFO] consul: adding WAN server whitehall.dc1 (Addr: 127.0.0.1:8300) (DC: dc1)
    2017/01/29 21:27:49 [ERR] agent: failed to sync remote state: No cluster leader
    2017/01/29 21:27:51 [WARN] raft: Heartbeat timeout reached, starting election
    2017/01/29 21:27:51 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state
    2017/01/29 21:27:51 [DEBUG] raft: Votes needed: 1
    2017/01/29 21:27:51 [DEBUG] raft: Vote granted from 127.0.0.1:8300. Tally: 1
    2017/01/29 21:27:51 [INFO] raft: Election won. Tally: 1
    2017/01/29 21:27:51 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2017/01/29 21:27:51 [INFO] raft: Disabling EnableSingleNode (bootstrap)
    2017/01/29 21:27:51 [DEBUG] raft: Node 127.0.0.1:8300 updated peer set (2): [127.0.0.1:8300]
    2017/01/29 21:27:51 [INFO] consul: cluster leadership acquired
    2017/01/29 21:27:51 [DEBUG] consul: reset tombstone GC to index 2
    2017/01/29 21:27:51 [INFO] consul: New leader elected: whitehall
    2017/01/29 21:27:51 [INFO] consul: member 'whitehall' joined, marking health alive
    2017/01/29 21:27:52 [INFO] agent: Synced service 'consul'
==> Newer Consul version available: 0.7.3
```

Note that Consul agent is running in a cluster, although a poor one. This cluster contains a single node. 

Furthermore, Consul has discovered and registered a service. It is Consul itself, which is a nice sanity check.


Running with a User Interface
---

Consul sports a nice user interface. Let's stop the current agent and restart it with the HTTP UI support.

```
./consul agent -dev -advertise=127.0.0.1 -ui
```

We are now able to visit **http://localhost:8500/ui** and observe the UI.

* on the **Services** tab, we see a single **consul** service
* on the **K/V** tab, we see an empty key/value database



Running full-fledged Server
---

In production and in the proper deployment, let's run Consul agent as a standalone server, with configuration persistence.

```
./consul agent -server -bootstrap -data-dir=/Users/novotnyr/Library/consul -advertise=127.0.0.1 -ui
```

Parameter meaning:

* `-server`. Agent is running as a **server**, that means it is holding cluster state and handles queries.
* `-bootstrap`. Agent elects himself as a cluster leader, being the first cluster node that is starter. Only one agent can be in the bootstrap mode. Otherwise there will be multiple cluster leaders, which leads to the instable cluster.
* `-data-dir`. A directory which will hold persistent data (K/V store, services) between restarts.


Registering Services
===

## Manual Service Registration

Some of the services need to be registered manually. Usually, this applies for **MySQL**, **ElasticSearch** and other services that are not Consul-aware.

The manual service registration is handled by Consul REST API. 

Let's execute HTTP PUT to the following URL:

```
http://localhost:8500/v1/agent/service/register
```

The payload should represent at least the service *name*, *address* and *port*.

```
{
  "Name": "mysql",
  "Address": "127.0.0.1",
  "Port": 3306
}
```

Upon completing of this HTTP request, the **mysql** service will appear in the **Services** tabs in the Consul UI.


Consul as a DNS Resolver
===

Consul is able to pose as a DNS resolver, thus heavily simplifying service discovery and resolution. With this facility, any client can resolve IP address of any service registered in consul.

We can ping a special `service.consul` domain:

```
ping mysql.service.consul
```

and retrieve the full IP of the MySQL service.


Consul as a DNS Resolver on MacOS
---

1. Install `dnsmasq` via Homebrew

   ```
   brew install dnsmasq
   ```

2. Use the instructions provided by Homebrew

   1. Copy the example configuration

   ```
    cp /usr/local/opt/dnsmasq/dnsmasq.conf.example /usr/local/etc/dnsmasq.conf
   ```

   1. Launch `dnsmasq` at startup:

      ```
      sudo cp -fv /usr/local/opt/dnsmasq/*.plist /Library/LaunchDaemons
      sudo chown root /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
      ```

   2. Launch `dnsmasq` now:

      ```
      sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
      ```


3. Configure `/etc/resolver`

   ```
   cat /etc/resolver
   # cat: /etc/resolver: No such file or directory
   sudo mkdir /etc/resolver
   ```

4. Create `service.consul` domain resolver file:

   ```
   sudo nano /etc/resolver/service.consul
   ```


5. Add the content:

   ```
   nameserver 127.0.0.1
   port 8600
   search service.consul
   ```

6. Check the functionality:
   1. `scutil —dns`. You should see `service.consul` resolver
   2. use `dig @127.0.0.1 -p 8600 mysql.service.consul ANY` 
   3. ping via `ping mysql.service.consul`


Consul on Windows
===

* Download Consul

* extract it to the directory

* setup the `PATH` environment variable so you can run the `consul` from commandline

* prepare config file

  * create `/etc/consul.d` (corresponding to the `C:\etc\consul.d`) with the following content:

    ```
    {
      "bootstrap" : true,
      "server" : true,
      "ui" : true,
      "service" : {
        "name" : "web",
        "port" : 80
      },
      "ports" : {
        "dns" : 53
      },
      "recursors" : [
        "8.8.8.8"
      ]
    }
    ```
  * the `web` service is just for testing purposes. It will be autoregistered at startup.   

* start Consul agent. Make sure that there is `C:/var/lib/consul` directory:

  ```
  consul agent -data-dir=/var/lib/consul -config-file=/etc/consul.d -bind=127.0.0.1 
  ```

* configure Windows network adapter:

  * open **Control Panel** | **Networks and Internet** | **Networks Center**

  * navigate to the active connection details

  * show properties of the **TCP/IPv4 (Internet Protocol Version 4)**

  * set DNS to:

    * preferred: 127.0.0.1
    * alternative: 8.8.8.8

  * recheck the internet connections

  * flush the DNS cache via

    ```
    ipconfig /flushdns
    ```

* ping the `web` service in the Consul:

  ```
  ping web.service.consul
  ```