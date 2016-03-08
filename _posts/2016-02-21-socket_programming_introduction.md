---
layout: post
title: "Socket Programming Basics: Introduction"
---

Socket programming allows an application to communicate over a network. This is the 
underlying mechanism of communication for applications like Web Browsers, WS FTP etc.
While sockets provide the underlying communication channel for data to be passed around,
it does not provide any rules as to *how* two computers on a network must interact. One
well-defined way of interaction between computers is called the **Client-Server Model**.

## Client Server Model

In this scenario, there are two kinds of players: *server* and *client*. The server is
considered to be the source of information which maybe of any kind - like web pages, audio
etc. The client is the player that is requesting some information from the server. The
server is setup to listen to requests on a particular port on the machine. When it receives
a valid request from a client on the port on which it is listening, it parses and processes
the request and sends an appropriate response. Once the message to the client is sent, the
server and client can close the connection.

## Transport protocols: TCP and UDP

These *transport layer* protocols determine the mechanics of the interation between a client
and a server. TCP stands for Transmission Control Protocol and UDP stands for Universal
Datagram Protocol and they  govern how the data is transported over a physical network,
hence the name "transport layer". TCP and UDP have their advantages and disadvantages and
to learn more checkout reference [2]. However, a fact relevant to the discussion you must
know is that TCP is connection-based, while UDP is connectionless. This essentially means
that a connection must be estabilished before sending any data via TCP transport, while 
a UDP transport does not require a explicit connection handshake between client and server.


## TCP vs UDP Client-Server connections

Lets look at the overall mechanism of TCP and UDP client-server interactions (images taken
from [3]):

**TCP connection**

![tcp_image](/assets/2016-02-21_tcp_client_server.png){:style="align: center; clear: all"}

**UDP connection**

![udp_image](/assets/2016-02-21_udb_client_server.png){:style="align: center; clear: all"}

First lets look at the similarities. In both cases, the client and the server must setup a 
socket and the server must bind it to a port. Similarly, in both cases, the connections must
be closed after the interaction is done.

Now in the case of TCP transport, the server must `listen()` to the port bound to a socket.
This allows the TCP server to listen for connection requests from clients (remember that
TCP requires a connection prior to data transfer). Following this, the requests from clients
start coming in and get queued up. The `accept()` call processes the next request for 
connection from a queue of pending requests. On the client side, the `connect()` call sends
a connection request to a server and waits for response. Once the connection is established,
the client and server can share data using `read()` and `write()` system calls.

The UDP interaction is much simpler - there is no need to setup a connection - hence no calls
to `listen()`, `accept()`, `connect()`. The data transmission and reception is done using the 
IP address of the recepient directly.

In the next post I would describe the specific functions used in Linux to setup sockets
and transfer data between a client and server.

## Reference
1. https://www.youtube.com/watch?v=eVYsIolL2gE&list=PL0JmC-T2nhdgJ2Lw5YdufR8MffaQdAvEf&index=1
2. https://www.youtube.com/watch?v=Vdc8TCESIg8
3. http://www.tenouk.com/Module39a.html
4. http://www.cs.dartmouth.edu/~campbell/cs60/socketprogramming.html
