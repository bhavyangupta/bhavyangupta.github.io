---
layout: post
title: "Socket Programming Basics: Hello World"
---

In this post I present some code for basic TCP / UDP server and clients.

## Source Code: Server

```
/**
 * @file server-tcp.c
 * @brief TCP server. Usage ./server-tcp 
 * @author Bhavya Narain Gupta
 * @version 0.0.0
 * @date 2016-02-21
 */
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<netinet/in.h>
#include<string.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/socket.h>
 
#define LISTENER_BACKLOG 5

int main(int argc, char* argv[]) {
  int socket_fd, client_fd;
  struct sockaddr_in addr_server, addr_client;
  socklen_t socket_size;
  char msg[] = "Hello World!!";
  char client_name[INET_ADDRSTRLEN];
  int n_sent = 0;

  memset(&addr_client, 0, sizeof(addr_client));
  memset(&addr_server, 0, sizeof(addr_server));
  socket_size = sizeof(addr_server);
  
  // Create socket
  socket_fd = socket(AF_INET, SOCK_STREAM, 0); // TCP socket
  if (socket_fd == -1) {
    perror("Socket error: ");
    exit(-1);
  }
  // Fill in server address
  addr_server.sin_family = AF_INET;
  addr_server.sin_port = htons(10000);
  addr_server.sin_addr.s_addr = htonl(INADDR_LOOPBACK); // localhost
  // Bind socket
  if (bind(socket_fd, (struct sockaddr* )&addr_server, socket_size) == -1) {
    perror("Bind error: ");
    exit(-1);
  }
  // Listen on socket
  if( listen(socket_fd, LISTENER_BACKLOG) == -1) {
    perror("Listen error: ");
    exit(-1);
  }
  while(1) {
    client_fd = accept(socket_fd, (struct sockaddr* )&addr_client, &socket_size); 
    if( client_fd == -1) {
      perror("Accept error: ");
      exit(-1);
    }
    // You can talk to the client now...
    n_sent = write(client_fd, msg, sizeof(msg));
    inet_ntop(AF_INET, &addr_client, client_name, socket_size);
    printf("Bytes sent: %i to %s \n", n_sent, client_name);

    close(client_fd);
  }
  // Sent Message
}
```

```
/**
 * @file server-udp.c
 * @brief UDP server. Usage: ./server-udp
 * @author Bhavya
 * @version 0.0.0
 * @date 2016-02-23
 */

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <errno.h>
#define PORT 10000
#define BUFFER_SIZE 255

int main(int argc, char* argv[]) {
  struct sockaddr_in addr_server, addr_client;
  int socket_fd;
  int rx_bytes;
  char ack[] = "Thank You";
  char buff[BUFFER_SIZE];
  socklen_t socket_size;
  memset(&addr_server, 0, sizeof(addr_server));
  socket_size = sizeof(addr_server);
  
  // Create a UDP socket
  socket_fd = socket(AF_INET, SOCK_DGRAM, 0);
  if (socket_fd == -1) {
    perror("Socket error: ");
    exit(-1);
  } 

  // Fill in the server socket address
  addr_server.sin_family = AF_INET;
  addr_server.sin_port = htons(PORT);
  addr_server.sin_addr.s_addr = htonl(INADDR_LOOPBACK); // localhost

  // Bind the socket to the address.
  if (bind(socket_fd, (struct sockaddr* ) &addr_server, socket_size) == -1) {
    perror("Bind error: ");
    exit(-1);
  } else {
    printf("Socket bound to port: %i \n", PORT);
  }
  // Wait for a request and send a hello world string
  while (1) {
    printf("Waiting on Port : %i \n", PORT);
    // This syscall fills in the empty addr_client struct that is passed for sending a reply
    // to a client.
    rx_bytes = recvfrom(socket_fd, buff, BUFFER_SIZE, 0, (struct sockaddr*) &addr_client, &socket_size );  
    printf("Read: %i, %s \n", rx_bytes, buff);
    if (rx_bytes > 1) {    
      sendto(socket_fd, ack, sizeof(ack), 0, (struct sockaddr*) &addr_client, socket_size);
    }
  }
}
```


## Source Code: Client

```
/**
 * @file client-tcp.c
 * @brief TCP client. Usage: ./client-tcp
 * @author Bhavya Narain Gupta
 * @version 0.0.1
 * @date 2016-02-21
 */

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<netinet/in.h>
#include<string.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/socket.h>
#define PORT 10000

int main(int argc, char* argv[]) {
  int socket_fd;
  struct sockaddr_in addr_server;
  socklen_t socket_size;
  char msg[] = "Ping";
  
  // Initialize server address struct
  memset(&addr_server, 0, sizeof(addr_server));
  socket_size = sizeof(struct sockaddr_in);

  // Create stream socket
  socket_fd = socket(AF_INET, SOCK_STREAM, 0);
  if (socket_fd == -1) {
    perror("Socket error: ");
    exit(-1);
  }

  // Fill in the address struct
  addr_server.sin_family = AF_INET;
  addr_server.sin_port = htons(PORT);
  addr_server.sin_addr.s_addr = htonl(INADDR_LOOPBACK); // Used for localhost

  // Request connection from server
  if (connect(socket_fd, (struct sockaddr*) &addr_server, socket_size) == -1) {
   perror("Connect error: ");
    exit(-1);
  }
  // The connect call blocks until the server responds.

  // Write to socket that is connected
  if (write(socket_fd, msg, sizeof(msg)) == -1) {
    perror("Write error: ");
    exit(-1);
  }
}
```

```
/**
 * @file client-udp.c
 * @brief UDP Client. Usage: ./client-udp
 * @author Bhavya
 * @version 0.0.0
 * @date 2016-02-23
 */

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <errno.h>
#define PORT 10000

int main (int argc, char* argv[]) {
  struct sockaddr_in addr_server;
  int socket_fd;
  char msg[] = "Ping";
  socklen_t socket_size;
  socket_size = sizeof(struct sockaddr);

  // Create datagram (UDP) socket 
  socket_fd = socket(AF_INET, SOCK_DGRAM, 0);
  if (socket_fd == -1) {
    perror("Socket error; ");
    exit(-1);
  }

  // Initalize the server address struct.
  memset(&addr_server, 0, sizeof(addr_server));
  addr_server.sin_family = AF_INET;
  addr_server.sin_port = htons(PORT);
  addr_server.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
  
  while(1) {
    // This system call sends data directly to a address.
    sendto(socket_fd, msg, sizeof(msg), 0, (struct sockaddr*)&addr_server, socket_size);
  }
}

```

