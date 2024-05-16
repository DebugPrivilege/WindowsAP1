# Summary

Windows Sockets defines how (Windows) network software should access network services, especially TCP/IP. It's a programming interface that facilitates communication between network software and network services, such as TCP/IP.

A socket is a communication endpoint — an object through which a Windows Sockets application sends or receives packets of data across a network. A socket has a type and is associated with a running process, and it may have a name.

Two socket types are available:

- **Stream sockets:**
Stream sockets provide for a data flow without record boundaries: a stream of bytes. Streams are guaranteed to be delivered and to be correctly sequenced and unduplicated.

- **Datagram sockets:**
Datagram sockets support a record-oriented data flow that is not guaranteed to be delivered and may not be sequenced as sent or unduplicated.
