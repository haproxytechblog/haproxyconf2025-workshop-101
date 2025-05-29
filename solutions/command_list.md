Use these commands to follow along with us as we walk through the workshop exercises.
# Setup
```
# clone the workshop repo
git clone https://github.com/haproxytechblog/haproxyconf2025-workshop-101.git

# execute all commands within the "workshop" directory
cd workshop

# bring up the containers
docker compose up -d

# check the logs for all the containers
docker compose logs

# go to this URL in your browser
http://localhost:80

# access the test servers (through HAProxy) using curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80

# check the logs for the HAProxy container
docker compose logs haproxy

# check the logs for all the containers
docker compose logs

# reload HAProxy
docker compose kill -s USR2 haproxy

# use socat to list the available HAProxy runtime API commands
echo "help" | docker run -i --net workshop_mynetwork --rm alpine/socat stdio TCP4:haproxy:9000
```

# Task 1:

```
# add the following to your haproxy.cfg:

frontend cart2 from tcp_defaults
  bind :81
  default_backend cartservers

# reload HAProxy
docker compose kill -s USR2 haproxy

# go to this URL in your browser
http://localhost:81/

# access the test servers (through HAProxy) using curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:81
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80

# Check the HAProxy logs
docker compose logs haproxy
```

# Task 2:

```
# add the following to your haproxy.cfg:
backend cartservers from http_defaults
  balance leastconn

# reload HAProxy
docker compose kill -s USR2 haproxy
```

# Task 3:

```
# add the following to your haproxy.cfg:
  server cart1 server1:8080 check
  server cart2 server2:8080 check
  server cart3 server3:8080 check

# reload HAProxy  
docker compose kill -s USR2 haproxy
```

# Task 4:

```
# add the following to your haproxy.cfg:
  acl evil_agent req.hdr(user-agent) -m sub evil
  http-request deny if evil_agent

# reload HAProxy
docker compose kill -s USR2 haproxy

# send a request using curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv -H "User-Agent: evil agent" haproxy:80
```

# Task 5:

```
# add the following to your haproxy.cfg:
  http-request set-var(txn.http_version) req.ver
  http-response add-header Via "%[var(txn.http_version)] %[hostname]"

# reload HAProxy
docker compose kill -s USR2 haproxy

# send a request with curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80

# check the HAProxy logs
docker compose logs haproxy
```

# Task 6:

```
# add the following to your haproxy.cfg:
  log-format "$HAPROXY_HTTP_LOG_FMT dbg={%[fs.debug_str,when(!normal)]}"

# send a request with curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv -H "User-Agent: evil agent" haproxy:80

# check the HAProxy logs
docker compose logs haproxy
```

# Task 7:

```
# add the following to your haproxy.cfg:
  use_backend %[path,map_beg(paths.map,cartservers)]

# reload HAProxy
docker compose kill -s USR2 haproxy

# send requests using curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80/cart/
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80/review/

# Check the HAProxy logs
docker compose logs haproxy
```

# Task 8:

```
# reload HAProxy
docker compose kill -s USR2 haproxy

# Run this several times until your requests are denied
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80
```

# Bonus Task 1:

```
# add the following to your haproxy.cfg:
  use_backend %[path,map_beg(opt@paths.map,cartservers)]

# reload HAProxy
docker compose kill -s USR2 haproxy  

# add an entry to the map using the HAProxy Runtime API
echo "add map paths.map /api/ reviewservers" | docker run -i --net workshop_mynetwork --rm alpine/socat stdio TCP4:haproxy:9000

# send a request using curl
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80/api/

# Check the HAProxy logs
docker compose logs haproxy

# use the HAProxy Runtime API to show all entries in the map
echo "show map paths.map" | docker run -i --net workshop_mynetwork --rm alpine/socat stdio TCP4:haproxy:9000
```

# Bonus Task 2:

```
# add the following to your haproxy.cfg:
frontend stats from http_defaults
  mode http
  bind :8404
  stats enable
  stats refresh 10s
  stats uri /stats
  stats show-modules

# reload HAProxy
docker compose kill -s USR2 haproxy

# go to this URL in your browser
http://localhost:8404/stats

# stop the server1 container
docker compose stop server1

# restart the server1 container
docker compose start server1
```

# Bonus Task 3:

```
# add the following to your haproxy.cfg:
backend reviewservers from http_defaults
  balance hash path
  hash-type consistent

# reload HAProxy
docker compose kill -s USR2 haproxy

# send several requests
docker run -ti --net workshop_mynetwork --rm alpine/curl-http3 curl -vv haproxy:80

# check the HAProxy logs
docker compose logs haproxy
```

