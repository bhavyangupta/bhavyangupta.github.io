---
layout: post
title: "Remote Procedure Calls using rpcgen"
---

In this post I want to talk about writing generating client server stubs for 
remote procedure calls using rpcgen. Although rpcgen supports many languages,
this post will focus on C/C++. I will first describe the motivation for
RPC's and then proceed to describe how they can be used in a C/C++ application.

## Motivation

Previously, I wrote about socket programming and how simple TCP/UDP clients 
and servers can be implemented using the sockets API, thereby allowing an 
application to communicate with a remote machine. This opens up a bunch of
possibilities. For example, this mechanism can be used to fetch files that are
stored on the server, perform computation or more generally, gain access to 
resources that are available to the server but not the client. 

This is the essence of **middleware** - providing access to resources that a 
local OS does not have.

Lets take the case of distributed computation as a working example to motivate
the need for RPC's. In such an application, the client machine does not have the
required computational power for carrying out a computation - say adding two 
numbers - and it wants a more powerful machine (Server) to do this computation 
for it.

Now if you were to use the bare sockets API to do this, you would have to
typically do the following on the server and client side:

1. Establish connection (if using TCP).
2. Marshal/Unmarshal the variable values.
3. Send/Receive data packets.t
4. Process the unmarshalled data.

While this may sound straight-forward for a simple `add` operation, it becomes
tideous for large chunks of heterogenous data. Directly dealing with low-level
sockets also makes the work of the application developer harder by diverting 
his/her attention to a detail irrelevant to the problem he/she cares about. What
is needed is a way to abstract out and automate the process of doing the
low-level communication and processing of data. This is the problem that RPCs
fix.

Remote procedure calls make procedures (or functions) that are part of the code
running on the server *look like* they are part of the client's code. This 
simplifies the work of the application developer who writes the client code, 
since he/she can write the code as if the procedure was implemented locally.
The low-level communication and the steps 1-4 listed above are carried out by
two *auto-generated* code fragments called the **client-stub** and 
**server-stub**. However, in order to generate this code fragments, the 
interface between the two must be well-defined and known. The interface is 
described using an **Interface Definition Language(IDL)** and it holds the 
information that is contained in a function prototype. This includes but is not
limited to the name of the procedure, type of arguments, return type etc.

How are the client and server stubs built? Here's where `rpcgen` comes in. `rpcgen`
is a "compiler" that reads an IDL file and generates the stubs for you. Note that
this compiler doesn't generate object code, rather it creates source code files
and Makefiles that you can modify and link with your main code.

This is all I can explain about RPC's here, but if you want to learn more about
the RPC mechanism refer to [1]. 

## Writing the interface

The first step in using RPC's is to define the *interface* between the server
and the client. For `rpcgen`, the IDL used is the XDR(External Data Representation)
format, with some upgrades. The RPC language specification extends the XDR
specification by adding `program` and `version` types. Note that in the .x files
you are defining data and not writing any code per-se. 

I am writing this post as I do my homework, so here's the IDL file I came up
with for a voting system that I have to implement:

```
const MAX_CANDIDATE_COUNT = 10;
const USERNAME_LENGTH = 10;
const PASS_LENGTH = 10;
const REPLY_LENGTH = 20;
const CANDIDATE_NAME_LENGTH = 20;
const CANDIDATE_COUNT_LENGTH = 10;
const CANDIDATE_ENTRY_LENGTH = 30;

typedef string reply_t<REPLY_LENGTH>;
typedef string name_t<CANDIDATE_NAME_LENGTH>;
typedef string entry_t<CANDIDATE_ENTRY_LENGTH>;
typedef name_t name_list_t<MAX_CANDIDATE_COUNT>;
typedef entry_t entry_list_t<MAX_CANDIDATE_COUNT>;
typedef string password_t<PASS_LENGTH>;
typedef string username_t<USERNAME_LENGTH>;

program VOTING_SYSTEM {
  version VER_1 {
    reply_t change_password(username_t username, password_t password) = 1;
    reply_t vote_zero() = 2;
    reply_t add_voter(int voter_id) = 3;
    reply_t vote(string candidate, int voter_id) = 4;
    name_list_t list_candidates() = 5;
    entry_list_t view_result(username_t username, password_t password) = 6;
  } = 1;
} = 0x20000199;

```
Let me explain some of the items above. The constants define limits of my "toy"
voting system. These are converted to `#define` statements by the `rpcgen` 
compiler. While the syntax looks very similar to C, there are some differences 
that I took some time to grasp. `rpcgen`s IDL *does not allow variable 
declarations*, therefore, to declare a variable length array of strings 
(`_list_t`), I had to first declare a new type `name_t` that defines a `string`
of a known max length, and then declare a new type `name_list_t` that defines a
variable-length array of the `name_t` type. This allowed me to pass and return 
variable-length arrays. Here's the specific piece of specification I am referring
to:

```
typedef string name_t<CANDIDATE_NAME_LENGTH>
typedef name_t name_list_t<MAX_CANDIDATE_COUNT>
```

The ` = 1` and other numbers after function prototypes are used to identify the 
remote procedures. A default function with `id = 0` is also provided with a 
`void` definition. The program version allows the client to identify the right
set of procedures and the version number allows the client to identify the
right *version* of the procedures. The version number is used when establishing
the connection to the server.

I would say its always a good practice to define the maximum length of strings/
variable-length arrays, otherwise the compiler assumes the maximum possible 
length (2^32 - 1). The problem with this is that while transferring data, the stubs 
zero-pad the variable length arrays until the padded-length is equal to the 
maximum length, so if you don't optimise the length you would be sending a bunch
of unnecessary zeros over the network.

## Building the stubs

Generating the stubs is easy:

```
rpcgen -NCa voting_system.x

# Flags:
# -N: use the C-style function declarations - allows functions with more than one
#      arguments in the IDl file
# -C: generate C code that can also be compiled using g++
# -a: generate all stubs, templates and makefiles
```

This generates the server/client stubs along with server/client templates that 
contain the `main` functions. A `Makefile` is also generated that can be used to 
build the files. 

## Modifying the stubs

The files generated won't really be usable. 

`static` - importance of it


## Run!




## Final output:

Right-click -> view image to maximise

![tty_gif](/assets/tty.gif){:style="align: center; clear: all"}


## Reference

1. [CIS 505 Spring 2016 notes](/assets/2016-03-08-lecture_notes.pdf)
2. http://www.shrubbery.net/solaris9ab/SUNWdev/ONCDG/p7.html
3. https://docs.oracle.com/cd/E19455-01/805-7224/6j6q44ci8/index.html
4. https://docs.oracle.com/cd/E19455-01/805-7224/6j6q44cia/index.html
5. http://www.cprogramming.com/tutorial/rpc/remote_procedure_call_start.html
6. https://docs.freebsd.org/44doc/psd/22.rpcgen/paper.pdf


<!--
- What is RPC ?
- Why do you need it?
- What is RPCGEN?
- Attach notes?


- how to go about writing it 
XDR specification
- It is not a programming language - it is a language to describe data.
- Zero padding to align to four-bytes
- For variable length data types, the actual length of the content is encoded 
  before the first byte of the data.
- always specify the maximum length - even for variable length types because
  otherwise it assumes the maximum value - would pad zeros and use a lot of 
  space in encoding.


-->