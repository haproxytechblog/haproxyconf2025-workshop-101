# HAProxyConf 2025 - HAProxy 101 workshop

A fun introduction to HAProxy!

## Prerequisite steps

1. Install Docker.

   * [Docker Desktop on Linux](https://docs.docker.com/desktop/setup/install/linux/)
   * [Docker Desktop on Mac](https://docs.docker.com/desktop/setup/install/mac-install/)
   * [Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)

1. Install git.

   * [Git on Linux](https://git-scm.com/downloads/linux)
   * [Git for Mac](https://git-scm.com/downloads/mac)
   * [Git for Windows](https://git-scm.com/downloads/win)

1. Install a text editor.

   * [Visual Studio Code](https://code.visualstudio.com/download) - Linux, Mac, Windows

1. Clone this repository with git.

# Lab Exercises

We will complete a series of configuration tasks for an HAProxy instance run in a Docker container. We will use [Docker Compose](https://docs.docker.com/compose/) to create the HAProxy container, to create test servers that HAProxy will connect to, and to define the necessary networking settings. To get HAProxy and the test servers up and running, follow these steps.

Note: depending on your system, you may need to append `sudo` before all `docker` commands.

1. Within the `workshop` directory, run the following Docker compose command:

   ```
   docker compose up -d
   ```

   This command uses the `compose.yaml` file in the `workshop` directory to create:

   * An HAProxy container running the latest version of [HAProxy](https://hub.docker.com/r/haproxytech/haproxy-alpine). The ports we will need for these exercises are exposed on the container.
   * Six containers each running a [jmalloc echo server](https://hub.docker.com/r/jmalloc/echo-server). This is a utility that receives messages from HTTP clients and returns in each responses the message it received in the request. It runs by default on port 8080.
   * A [network bridge](https://docs.docker.com/engine/network/drivers/bridge/) that allows our HAProxy instance to communicate with our test servers.


   Output:

   ```
   [+] Running 8/8
    ✔ Network workshop_mynetwork    Created   0.1s 
    ✔ Container workshop-haproxy-1  Started   1.8s 
    ✔ Container workshop-server1-1  Started   1.3s 
    ✔ Container workshop-server2-1  Started   1.0s 
    ✔ Container workshop-server3-1  Started   1.0s 
    ✔ Container workshop-server4-1  Started   1.8s 
    ✔ Container workshop-server5-1  Started   1.8s 
    ✔ Container workshop-server6-1  Started   1.5s
   ```

   HAProxy and the test web servers are now up and running. You can use `docker compose logs` to see their logs. This output includes the logs from all of the containers created using the Docker compose file:

   ```
   docker compose logs
   ```

   Output:
   ```
   server1-1  | Echo server listening on port 8080.
   server5-1  | Echo server listening on port 8080.
   haproxy-1  | [NOTICE]   (1) : Initializing new worker (8)
   haproxy-1  | [NOTICE]   (1) : Loading success.
   server6-1  | Echo server listening on port 8080.
   server3-1  | Echo server listening on port 8080.
   server2-1  | Echo server listening on port 8080.
   server4-1  | Echo server listening on port 8080.
   ```

   If no errors are present, HAProxy is online and working.

   If you encounter the following, it is because the HAProxy container came online before the test servers. Wait a moment and HAProxy will show the servers online:

   ```
   haproxy-1  | [WARNING]  (8) : Server reviewservers/review1 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 2 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.

   haproxy-1  | [WARNING]  (8) : Server reviewservers/review1 is UP, reason: Layer4 check passed, check duration: 0ms. 3 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
   ```

   If for some reason you want to delete and recreate these containers you can run `docker compose down` inside the `workshop` directory. You can then run `docker compose up -d` again to create the containers again. 

1. In the `workshop` directory there is a file named `haproxy.cfg`. This is the file we will use for our HAProxy configuration. Our Docker compose defintion for the HAProxy container mounted the `workshop` directory to our HAProxy docker container at `/usr/local/etc/haproxy`, so all files in this directory are also inside the container. This is where HAProxy will look for its files.

1. Access the container on port 80 in the browser. One of the jmalloc echo servers will handle the request and send back a response:

   ```
   Request served by e3abd0c22612

   GET / HTTP/1.1

   Host: 172.31.26.90
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
   Accept-Encoding: gzip, deflate
   Accept-Language: en-US,en;q=0.9
   Connection: close
   Upgrade-Insecure-Requests: 1
   User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
   X-Forwarded-For: 172.18.0.9
   ```

1. View the logs to see more details about the request as it passed through HAProxy:

   ```
   docker compose logs haproxy
   ```

   Output:

   ```
   haproxy-1 | 172.18.0.9:59639 [19/May/2025:20:18:27.263] cart cartservers/cart1 0/0/0/4/4 200 589 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
   ```

   We can see that the `cart` frontend caught the request, as it was listening on port 80, and the server named `cart1` in the backend `cartservers` ultimately received the request and responded.


1. Alternatively, we can issue a similar command via `curl` by running a [curl Docker container](https://hub.docker.com/r/alpine/curl-http3) and passing a command to it. The container exits after the command completes:

   ```
   docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80
   ```

   Since we are running the curl container in the same network (`workshop_mynetwork`) as the HAProxy container, we can refer to the HAProxy container by name in the URL we pass to curl, in this case `haproxy`. 

   As we have the verbosity high for testing (`-vv`), we can see the exchange between `curl` and HAProxy:

   ```
   20:21:50.550914 [0-0] * Host haproxy:80 was resolved.
   20:21:50.551075 [0-0] * IPv6: (none)
   20:21:50.551301 [0-0] * IPv4: 172.18.0.8
   20:21:50.551328 [0-0] * [SETUP] added
   20:21:50.551369 [0-0] *   Trying 172.18.0.8:80...
   20:21:50.551534 [0-0] * Connected to haproxy (172.18.0.8) port 80
   20:21:50.551555 [0-0] * using HTTP/1.x
   20:21:50.551601 [0-0] > GET / HTTP/1.1
   20:21:50.551601 [0-0] > Host: haproxy
   20:21:50.551601 [0-0] > User-Agent: curl/8.11.0-DEV
   20:21:50.551601 [0-0] > Accept: */*
   20:21:50.551601 [0-0] > 
   20:21:50.551716 [0-0] * Request completely sent off
   20:21:50.553203 [0-0] < HTTP/1.1 200 OK
   20:21:50.554155 [0-0] < content-type: text/plain
   20:21:50.554664 [0-0] < date: Mon, 19 May 2025 20:21:50 GMT
   20:21:50.555168 [0-0] < content-length: 148
   20:21:50.555459 [0-0] < 
   Request served by 10eedea49a9e

   GET / HTTP/1.1

   Host: haproxy
   Accept: */*
   Connection: close
   User-Agent: curl/8.11.0-DEV
   X-Forwarded-For: 172.18.0.9
   20:21:50.555627 [0-0] * Connection #0 to host haproxy left intact
   ```

   The logs show that this time, the server named `cart3` in the backend `cartservers` ultimately handled this request.

   ```
   docker compose logs haproxy
   ```

   ```
   haproxy-1  | 172.18.0.9:42568 [19/May/2025:20:21:50.551] cart cartservers/cart3 0/0/0/0/0 200 254 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
   ```

1. In the `workshop` directory, we can run `docker compose logs` to see the test web server logs. We can see our GET requests:

   ```
   docker compose logs
   ```

   Output:

   ```
   server4-1  | Echo server listening on port 8080.
   server1-1  | Echo server listening on port 8080.
   server1-1  | 172.18.0.8:39206 | GET /
   server5-1  | Echo server listening on port 8080.
   server6-1  | Echo server listening on port 8080.
   server3-1  | Echo server listening on port 8080.
   server3-1  | 172.18.0.8:42742 | GET /
   server2-1  | Echo server listening on port 8080.
   server2-1  | 172.18.0.8:44130 | GET /favicon.ico
   ```

1. For these tasks, we will not have to completely restart HAProxy, nor will we have to delete and recreate our container to change the configuration. We can instead make our changes to our local copy of the `haproxy.cfg` file and perform a hitless reload of HAProxy using this command:

   ```
   docker compose kill -s USR2 haproxy
   ```

1. After running this command, check the HAProxy logs again. We see that HAProxy reloads:

   ```
   haproxy-1  | [NOTICE]   (1) : Reloading HAProxy
   haproxy-1  | [NOTICE]   (1) : Initializing new worker (10)
   haproxy-1  | [NOTICE]   (1) : Loading success.
   haproxy-1  | [WARNING]  (8) : Proxy cart stopped (cumulated conns: FE: 2, BE: 0).
   haproxy-1  | Proxy cart stopped (cumulated conns: FE: 2, BE: 0).
   haproxy-1  | Proxy cartservers stopped (cumulated conns: FE: 0, BE: 2).
   haproxy-1  | Proxy reviewservers stopped (cumulated conns: FE: 0, BE: 0).
   haproxy-1  | [WARNING]  (8) : Proxy cartservers stopped (cumulated conns: FE: 0, BE: 2).
   haproxy-1  | [WARNING]  (8) : Proxy reviewservers stopped (cumulated conns: FE: 0, BE: 0).
   haproxy-1  | [NOTICE]   (1) : haproxy version is 3.1.7-c3f4089
   haproxy-1  | [WARNING]  (1) : Former worker (8) exited with code 0 (Exit)
   ```

With all of this in place, we are ready to move on to our exercises!

## Task #1: Add a new frontend that binds on port 81

Task: Create a new frontend named `cart2` that inherits from the defaults section named `tcp_defaults`. The frontend should bind on port 81 and should forward requests to the `cartservers` backend.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Frontend configuration example](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/frontends/#frontend-configuration-example)

Testing: Send a request to HAProxy in the browser or using curl on port 81.

```
http://localhost:81/
```

```
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:81
```

We set `option tcplog` in the `tcp_defaults` defaults section that the `cart2` frontend inherits from. This sets the logging format. The other frontend `cartservers` which inherits from the `http_defaults` section uses the `option httplog` log format. Check the logs to see how requests to each frontend are logged.

`cart` (`option httplog`):
```
haproxy-1  | 75.201.195.235:64258 [20/May/2025:17:04:26.619] cart cartservers/cart1 0/0/0/0/0 200 589 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1" dbg={-}
```

`cart2` (`option tcplog`):
```
haproxy-1  | 75.201.195.235:64260 [20/May/2025:17:04:29.365] cart2 cartservers/cart1 0/0/124 536 -- 4/2/0/0/0 0/0
```

The purpose of this task was to show multiple frontends inheriting from different defaults sections. For more information about logging formats, see [Set the log format](https://www.haproxy.com/documentation/haproxy-enterprise/administration/logs/#set-the-log-format).

## Task #2: Change the load balancing algorithm

Task: Change the load balancing algorithm for the backend named `cartservers` from `roundrobin` to `leastconn`.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Change the load balancing algorithm](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/backends/#change-the-load-balancing-algorithm)

Testing: Send some requests to HAProxy in the browser or using curl. You may not see any change in behavior, as the servers are not under heavy load.

## Task #3: Enable health checks

Task: Enable health checks on the servers listed in the backend named `cartservers`.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Enable health checks](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/backends/#enable-health-checks)

Testing: We will come back to testing this one later (*Bonus Task #2: Enable the statistics dashboard*).

## Task #4: Use an ACL to deny requests

Task: In the frontend named `cart`, add an ACL that will deny requests with a User-Agent containing the substring "evil".

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Block requests](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/acls/#block-requests)

Testing: Test with curl.

```
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv -H "User-Agent: evil agent" haproxy:80
```

Your request will be denied:

```
<html><body><h1>403 Forbidden</h1>
Request forbidden by administrative rules.
</body></html>
```

## Task #5: Capture HTTP version and hostname in a variable

Task: In the frontend named `cart`, create a variable that will hold the HTTP version of the request. Populate the `Via` header on the response with the HTTP version and the hostname.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Fetches](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/fetches/)

Testing: Access HAProxy in the browser on port 80 or using curl. The Via header will be populated with the HTTP version and the hostname of the server.

Output from curl:
`21:25:08.176388 [0-0] < via: 1.1 b17607f87f9d`

In our case, the hostname is the container ID of the HAProxy Docker container (`b17607f87f9d`).

## Task #6: Use the when converter to log conditionally

Task: Use the `when()` converter to log debug information when processing encounters an error.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Conditionally log debug_str](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/converters/#example-%231%3A-conditionally-log-debug_str)

Testing: Use the same curl command you used for Task #4. HAProxy will log the debug string.

```
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv -H "User-Agent: evil agent" haproxy:80
```

Output in the logs:

```
haproxy-1  | 172.18.0.9:34060 [19/May/2025:21:23:18.614] cart cart/<NOSRV> 0/-1/-1/-1/0 403 192 - - PR-- 1/1/0/0/0 0/0 "GET / HTTP/1.1" dbg={ h1s=0x7e18d4d50080 h1s.flg=0x14010 .req.state=MSG_DONE .res.state=MSG_DONE .meth=GET status=403 .sd.flg=0x40404601 .sc.flg=0x00034402 .sc.app=0x7e18d4cdd400 .subs=0 h1c.flg=0x0 .sub=0 .ibuf=0@0+0/0 .obuf=0@0+0/0 .task=0x7e18d4d44ae0 .exp=<NEVER> conn.flg=0x80000300}
```

## Task #7: Use a map to implement path-based routing

Task: Use a map file to enable path based routing for the frontend named `cart`. Requests to `/cart/` should be served by the backend named `cartservers`. Requests to `/review/` should be served to the backend named `reviewservers`. The required map file is located in the the `workshop` directory and is mounted into HAProxy Docker container.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Map files for path-based routing](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/map-files/#map-files-for-path-based-routing)

Testing: Access HAProxy in the browser or with curl. Go to `/cart/` and `/review/`. (Tip: Remember to include the forward slash on the end of the path). Each of these requests should route to its appropriate backend. Look in the logs to see how the requests are handled:

```
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80/cart/
```

```
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80/review/
```

```
haproxy-1  | 172.16.0.2:46562 [20/May/2025:18:55:21.791] cart cartservers/cart1 0/0/0/1/1 200 259 - - ---- 1/1/0/0/0 0/0 "GET /cart/ HTTP/1.1" dbg={-}
haproxy-1  | 172.16.0.2:47456 [20/May/2025:18:55:34.790] cart reviewservers/review1 0/0/0/1/1 200 261 - - ---- 1/1/0/0/0 0/0 "GET /review/ HTTP/1.1" dbg={-}
```

## Task #8: Uncomment the stick table example

Task: Uncomment the stick table example to see stick tables in action.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Create a stick table](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/stick-tables/#create-a-stick-table)


Testing: Send several requests in a short time frame. Your requests will eventually be rejected, as they exceed the request threshold specified in the stick table.

```
403 Forbidden
Request forbidden by administrative rules.
```

Be sure to comment out the stick table example again and reload HAProxy before proceeding with the next tasks.

## Bonus Task #1: Use the Runtime API to update routing rules

Task: Change the `paths.map` file to be an *optional* file which the load balancer will load from disk on startup and reload HAProxy. Use the runtime API to add an additional entry to the file which will route requests with `/api/` in the path to the `reviewservers` backend — do not reload HAProxy after executing the Runtime API command.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Virtual and optional files](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/map-files/#virtual-and-optional-files)

Testing:

We can access HAProxy's Runtime API using a [Docker container running socat](https://hub.docker.com/r/alpine/socat).

Use the runtime API command `add map` to add an entry for the `/api/` path to the map (copy/paste both lines):

```
echo "add map paths.map /api/ reviewservers" |
docker run -i --net workshop_mynetwork --rm alpine/socat stdio TCP4:haproxy:9000
```

Test in the browser or using curl, accessing the `/api/` path.

Check the logs to see that the request to `/api/` was routed to the `reviewservers` backend:

```
haproxy-1  | 172.18.0.9:50036 [19/May/2025:21:40:41.524] cart reviewservers/review3 0/0/0/1/1 200 258 - - ---- 1/1/0/0/0 0/0 "GET /api/ HTTP/1.1" dbg={-}
```

You can use the `show map` Runtime API command to see all of the entries in the map (copy/paste both lines):

```
echo "show map paths.map" |
docker run -i --net workshop_mynetwork --rm alpine/socat stdio TCP4:haproxy:9000
```

Output:

```
0x7e18d4d2b550 /cart/ cartservers
0x7e18d5242400 /review/ reviewservers
0x7e18d4db3e60 /api/ reviewservers
```

Note that since we used the map as an optional file and added an entry to it programmatically, this change is not saved to disk and will disappear upon restart. See the documentation reference for this task for more information about persisting these changes.

Issue the `help` command to see all available HAProxy Runtime API commands. Copy and paste both lines:

```
echo "help" |
docker run -i --net workshop_mynetwork --rm alpine/socat stdio TCP4:haproxy:9000
```

Note that for demonstration purposes and for ease of use in this workshop, we are using an additional container to run `socat`. For most real-world applications, you would probably connect to the HAProxy Runtime API in a different way, such as by running a `socat` binary installed on your system or by writing a program to issue the commands and process the output.

## Bonus Task #2: Enable the statistics dashboard

Task: Enable the statistics dashboard. Hint: the frontend you create should inherit from the `http_defaults` defaults section.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: [Enable the dashboard](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/alerts-and-monitoring/statistics/)

Testing: Go to `http://localhost:8404/stats` in the browser. Note: be sure that you have completed Task #3 which enables health checks for the servers.

Stop the `server1` test server using the following command:

```
docker compose stop server1
```

Output:
```
[+] Running 1/1
 ✔ Container workshop-server1-1  Stopped
```

What happens on the statistics dashboard?

Bring the test server back up:

```
docker compose start server1
```

We see the changes in the statuses of the servers because we enabled the health checks on the servers.

## Bonus Task #3: Use a hash-based load balancing algorithm

Task: Change the load balancing algorithm of the `reviewservers` backend to use the `hash` algorithm with the `path` sample fetch method. Set `hash-type` to `consistent`.

Reload HAProxy after making configuration changes:

```
docker compose kill -s USR2 haproxy
```

Documentation reference: 
[balance reference](https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/#4.2-balance)
[hash-type reference](https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/#4.2-hash-type)
[Fetch methods](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/fetches/)

Testing:

You should see in the logs that the same server receives all of the requests, in this case, `reviewservers/review3`:

```
haproxy-1  | 172.16.0.2:35334 [20/May/2025:19:21:18.501] cart reviewservers/review3 0/0/0/0/0 200 261 - - ---- 1/1/0/0/0 0/0 "GET /review/ HTTP/1.1" dbg={-}
haproxy-1  | 172.16.0.2:35342 [20/May/2025:19:21:22.408] cart reviewservers/review3 0/0/2/1/3 200 261 - - ---- 1/1/0/0/0 0/0 "GET /review/ HTTP/1.1" dbg={-}
haproxy-1  | 172.16.0.2:33216 [20/May/2025:19:21:42.332] cart reviewservers/review3 0/0/0/0/0 200 261 - - ---- 1/1/0/0/0 0/0 "GET /review/ HTTP/1.1" dbg={-}
```

