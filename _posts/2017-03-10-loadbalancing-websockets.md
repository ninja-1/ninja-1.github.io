---
layout: post
title: Load-balancing WebSockets
---

WebSockets are a relatively new technology, and as with most new technology there are more hurdles to jump to get everything running smoothly.

Recently, I have had the task of load-balancing multiple WebSocket servers in a way that maintains user sessions in the case of a disconnect. After some research, the most popular option for this task seemed to be HAProxy, so that’s what I went with. Lets get into the details…

## The Application

I’m using a Socket.IO-based connection. [The Socket.IO team have a nice walkthrough](https://socket.io/get-started/chat/), but just for reference, here’s a basic Socket.IO server. You’ll need two of them running to see what HAProxy can do.

```javascript
// server.js

const server = require('http').createServer()
const io = require('socket.io')(server)
 
// set this to something different for each server
const sID = process.env.SERVER_ID
io.on('connection', (socket) => {
  console.log(`connection on server ${sID}`)
})
 
const port = process.env.PORT || 3000
http.listen(port, () => {
  console.log('listening on *:3000')
})
```

## HAProxy

Here’s the part we’re going to focus on. We’re going to set up HAProxy to load-balance our WebSocket servers, but since we don’t want the user to lose their session data, and we don’t want to have the extra overhead from using something like Redis to store our session data, we’re going to implement cookie-based sticky sessions.


```
# haproxy.cfg

global
        log 127.0.0.1   local0
        log 127.0.0.1   local1 notice
        maxconn 4096
        user haproxy
        group haproxy
        daemon
 
defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        option forwardfor
        option http-keep-alive
        stats enable
        stats auth someuser:somepassword
        stats uri /haproxyStats
        timeout tunnel 2h
        timeout client 30s
        timeout server 30s
        timeout connect 5s
 
frontend http-in
        bind *:8080

        acl host_socket hdr(host) -i myserver.example.com

        use_backend socket_servers if host_socket
 
backend socket_servers
        balance roundrobin
        option forwardfor
        cookie SERVERID insert indirect nocache
        server node1 127.0.0.1:3000 check cookie bridge1
        server node2 127.0.0.1:3001 check cookie bridge2
```

Since we setup the stats options in haproxy.cfg we can login with someuser:somepassword to [http://localhost:8080/haproxyStats](http://localhost:8080/haproxyStats) and see that both of our WebSocket servers are (hopefully) up and can be reached.

Now when you point your Socket.IO client to HAProxy (which is running on port 8080 if you left the default config), you should be able to check the logs and see which of your servers the client has connected to, and you should also be able to check your browser and see that HAProxy has set a cookie named SERVERID with a value of node1 or node2.

On subsequent visits, HAProxy will forward you to the same server based on the cookie that was set the first time until the cookie expires.
