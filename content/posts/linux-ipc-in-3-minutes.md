+++
title = "Linux IPC in 3 minutes"
description = """Quick summary of Linux IPC mechanisms."""
draft = false
date = "2021-05-24"
author = "Mani Kumar"
categories = ["notes", "linux", "ipc"]
tags = ["linux", "ipc"]
+++

Linux IPC (Inter Process Communication)
---------------------------------------

IPC mechanisms are used to synchronize (sync) the processes and threads in
linux.

IPC mechanisms in linux are:

* Memory based: Shared variables, memory and regular files.
* Channel based: Pipes and message queues
* Stream based: Sockets

Memory based IPC
----------------

### Shared files and variables

The regular files on linux can be used to communicate between processes.
A race condition might arise when multiple processes tries to access at the
file at exact same time. This is prevented by using locks on the file.
The linux provides two lock APIs, `exclusive lock` and `shared lock`.

* Exclusive lock: A process that writes to the file must gain an exclusive
  lock before writing. An exclusive lock can be held only by one process at
  most, this prevents a race condition because no other process can access
  the file until the lock is released.
* Shared lock: A process that reads the file should gain at least a shared
  lock before reading. Multiple processes can hold a shared lock for reading
  the file. Any process intend to write to the file must wait until all the
  shared locks are released.

This locking mechanism applies to shared variables too. The only downside to
this IPC mechanism is that file access and locking are slow operations. Hence
not suitable for high performance communication.

### Shared Memory

Shared memory is designed to be used in large memory sharing between
processes. It is faster than file based sharing because of memory based data
access instead of file. Linux provides two separate APIs for shared memory:
System V (read as System Five) and POSIX. These two APIs are very different
and hence should not mixed in an application.

`Semaphores` are used to sync the shared memory access between processes.
A semaphore is a signalling mechanism and `mutex` is a locking mechanism.
Both are used to sync the shared memory access. A mutex is always first taken
and then released. Semaphores are used to either signal or wait and not both.

Channel based IPC
-----------------

Channels connect processes for communication. A channel has a write end and a
read end. It follows FIFO (first in, first out) order when writing or reading
bytes. One process writes to the channel at the write end and a different
process reads from this same channel at the read end. `Pipes` and
`message queues` are the channel based IPC mechanisms in linux.

### Pipes

Pipes can be named or unnamed. They can be used either interactively from the
command line or within programs. Pipes have strict FIFO behavior, the first
byte written is the first byte read and then the second byte, third byte
and so on.

### Message queues

Message queues are also FIFO based channel, but are flexible. Messages in the
queue can be accessed out of FIFO order by using the message type of the
message. A message queue is a sequence of messages and each message has two
parts: `payload` to store the message and `type` for message retrieval.
By using the message type the messages can be retrieved in any order from the
message queue.

The pipes and message queues are fundamentally unidirectional, one process
writes and another reads. For bidirectional communication it is better to use
stream based IPC.

Stream based IPC
----------------

Most applications often deal with streaming data with very large size. Shared
files or memory is not suited for this purpose. Pipes can be used for
streaming large data but this is generally unidirectional. For bidirectional
streaming sockets are used in linux.

### Sockets

Sockets are available in linux as:

* IPC sockets: used for communication between processes on the same host.
  Uses a local file as a socket address.
* Network sockets: used for communication between processes on different
  hosts. Network sockets need support from an underlying network protocol
  such as TCP or UDP.

The `IPC socket` and the `Network socket` are same at the API with differences
in the internal implementation.

Sockets configured as streams are bidirectional and control follows a
client-server pattern. The server listens on a specific host and port.
The clients try to connect to the server. On a successful connection the
requests from client and the corresponding responses from server then can flow
through the channel until it is closed on either end. A new connection is
required to be established when previous connection is closed to communicate
further.

References
----------

For more information and code examples refer to below links.

* [wiki/Inter-process communication][1]
* [Inter-process communication in Linux: Sockets and signals][2]
* [Mutexes and Semaphores Demystified][3]

[1]: https://en.wikipedia.org/wiki/Inter-process_communication
[2]: https://opensource.com/article/19/4/interprocess-communication-linux-networking
[3]: https://barrgroup.com/embedded-systems/how-to/rtos-mutex-semaphore
