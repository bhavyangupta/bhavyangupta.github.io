---
layout: post
title: "Socket Programming Basics: Important Functions"
---

## Functions used in socket programming

Before proceeding to a working example  describing how a client and server interact,
I would like to describe some of the basic functions used to setup connections and exchange data:

1. `int socket(int domain, int type, int protocol);`
    - This syscall creates a socket instance and returns a file descriptor corresponding to
      the socket. Note that in Linux writing to sockets is the same as writing to any other
      file.
    - For the current discussion, ignore all other parameters except the second one: `type`
      determines whether the socket will be a TCP socket(SOCK_STREAM) or UDP(SOCK_DGRAM).
    - It returns -1 if the socket could not be created due to some reason.

2. `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`
    - This syscall binds a socket (indicated by the file descriptor) to a specific IP
      address and port number.
    - The information regarding the socket address is passed via a pointer to a sockaddr
      type which must be created by you while programming. I will describe this structure
      later in the post.

3. `int listen(int sockfd, int backlog);`
    - This syscall listening for requests on the **TCP** socket specified by the file descriptor.
    - The `backlog` parameter determines the "queue size" for the server requests. Set this
      to the number of requests you want to queue while you are processing a given request.
      If the number of requests made to your server exceeds this number, the clients are
      sent a "Connection Refused" error message.
    - Note that this syscall is only used for connection-based sockets (like TCP). 

4. `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`
    - This syscall is used in **TCP** sockets to process the next client connection request
      and connects the socket to the client.
    - The address information of the client is written to the struct pointed by the `addr`
      parameter.
    - Note that `addrlen` must be initialised to the size of the structure pointed to by
      `addr`.

5. `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`
    - This syscall is used by a **TCP** client to request connection from a server. 
    - The address of the server is passed via a pointer to the `sockaddr` storing the 
      server address. It is your job to fill the address before sending it to `connect`.

6. `read(), write()`
   - These work as normal file read/write system calls and are used by the client and 
     server to exchange data.

7. `close(int fd)`
   - This closes the socket associated with a file descriptor.

8. `ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, 
                      struct sockaddr  *src_addr, socklen_t *addrlen);`
   - This syscall has to be used in **UDP** or conection-less sockets to get messages
     from a client. The address of the client is passed via the `src_addr` parameter.
   - Note that these can also be used for connection-oriented sockets in which case it
     is similar to the `read` call.

9. `ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, 
                  const struct sockaddr *dest_addr, socklen_t addrlen);`
   - This syscall has to be used in connection-less sockets to send messages. Although
     it can be used for connected sockets, in which case it is similar to the `write`
     syscall.

10. `htonl(uint32_t); ntohl(uint32_t)`
   - These are **endianness conversion** functions that are used to convert a variable
     stored in host endianness to the network endianness (htonl), and back (ntohl).
   - The 'l' at the end indicates that these are for `long` datatype and there are other
     variants.
   - Endianness conversion is neccessary for exchanging the addresses between client
     and server which may have different processors and memory representation for the same
     number.
   - The network endianness has been fixed to be Big Endian and so conversion takes place 
     *to* big endian before transmitting and *from* big endian while receiving.

11. `inet_aton(); inet_ntoa()`
   - Used to convert ASCII address to binary format(inet_aton) and back (inet_ntoa).


