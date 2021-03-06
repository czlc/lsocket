# README #

## lsocket ##

A library that provides network programming support for Lua.

Author: Gunnar Z?tl, 2013-2015.  
Released under the terms of the MIT license. See file LICENSE for details.

## Introduction ##

lsocket provides not complete, but good enough support for socket
programming. It supports IPv4, IPv6 and unix domain sockets, and
selects automatically which one to use based on the address you bind or
connect to. Also, it is almost-nonblocking: except for lsocket.select()
and nameserver lookups, nothing ever blocks. The functions connect, bind,
resolve and the method sendto transparently use name service resolution,
which may block, if a name server is not available or slow to respond.
However, name service resolution is not used if you pass an IP address
(IPv4 or IPv6) or a path (unix domain) as address argument to those
functions. And you can also make lsocket.select() nonblocking by passing
0 as a timeout value.

lsocket has been tested with lua 5.1.5, 5.2.4, 5.3.0 and luajit 2.0.1
on Linux and Mac OS X.

## Installing ##

This uses only stuff that comes with your system. Normally, calling

	sudo luarocks install lsocket

or when you have an unpacked source folder,

	sudo luarocks make

should do the right thing.

There is also a Makefile in the distribution directory, which has been
created for and on Linux and Mac OS X. luarocks uses this Makefile to
build lsocket.

## Using ##

Load the module with:

	lsocket = require "lsocket"

### Constructors ###

`socket = lsocket.bind( [type], [address], [port], [backlog] )`
:	creates a new socket and binds it locally.

	**Arguments**

	* `type`: (optional) type of socket to create, may be "udp", "tcp",
	  or "mcast", defaults to "tcp". If the type is mcast, the socket will
	  be a udp socket that is additionally set up for use as a ipv4
	  broadcast or ipv6 multicast client socket. mcast is not available
	  for unix domain sockets.
	* `address`: (optional) ip address or hostname to bind to, defaults to
	  lsocket.INADDR_ANY ('0.0.0.0'). If this is an IPv4 address, the
	  socket will be an IP socket, if it is an IPv6 address, the socket
	  will be an IPv6 socket, and if it contains a slash (/) character,
	  it will be a unix domain socket with the address as the path on
	  the file system. On linux, if the first char is '@', then it will
	  be a unix domain socket with an abstract socket name, i.e. one that
	  does not exist in the file system.
	* `port`: port number to bind to, meaningless with unix domain sockets.
	* `backlog`: (optional) length of connection backlog on this socket,
	  defaults to 5. This is only useful for tcp sockets.
 
	Returns the bound socket, or nil+error message on error.

`socket = lsocket.connect( [type], address, port, [ttl] )`
:	connects to a remote socket.

	**Arguments **

	* `type`: (optional) type of socket to create, may be "udp", "tcp",
	  or "mcast", defaults to "tcp". If the type is mcast, the socket will
	  be a udp socket that is additionally set up for use as a ipv4
	  broadcast or ipv6 multicast server socket. mcast is not available for
	  unix domain sockets.
	* `address`: ip address or hostname to connect to. See the address
	  parameter to bind() above for a more detailed explanation.
	* `port`: port number to connect to, meaningless with unix domain
	  sockets.
	* `ttl`: (optional) ttl for multicast packets, defaults to 1. Only
	  useful if type is "mcast".

	Returns the socket in connecting state, or nil+error message on error.
	Note that you need to select() the socket for writing to wait for the
	connect to finish. Of course you can also select the socket for
	reading, but if you select for writing, select() will return as soon
	as the socket is connected, whereas if you select for reading, select()
	will return when the socket is connected and the server has sent data.
	In any case select() will return if the connection fails. You should
	call status() on a socket that was created by connect() after it is
	first returned from select() in order to see whether the connection
	was successful.

### Socket Methods ###

`tbl = socket:info( [what] )`
:	returns a table with information about the socket.

	** Arguments **

	* `what`: (optional) specify what info you are interested in. The
	  result is returned in a table.

	Returns a table with the requested information:

	* If what is omitted or nil, return a standard set of socket infos.
	  These fields are in the table:
		* `fd`: socket file descriptor
		* `family`: ip protocol family, "inet" or "inet6"
		* `type`: "udp" or "tcp"
		* `listening`: true if the socket is listening (created by
		  lsocket.bind), false otherwise
		* `multicast`: true if the socket is a multicast socket, false
		  otherwise.
	* If what is "peer", return information about the socket peer. If
	  called on unix domain sockets, this will only return useful information
	  for sockets created with lsocket.bind(). These fields are in the table:
		* `family`: ip protocol family, "inet" or "inet6"
		* `addr`: ip address
		* `port`: port number
	* If what is "socket", return information about the local socket.
	  Fields as for "peer". If called on unix domain sockets, this will
	  only return useful information for sockets created with lsocket.connect().

`ok, err = socket:status()`
:	check a sockets error status.

	Returns true if the socket has no errors, nil + error message otherwise.

`fd = socket:getfd()`
:	Returns the sockets file descriptor.

	You probably don't need this, it is only for interaction with other packages.

`ok, err = socket:setfd(newfd)`
:	sets a sockets file descriptor. Only -1 (invalid descriptor) is
	allowed as argument. You probably don't need this, it is only
	for interaction with other packages.

	Returns true if the descriptor was -1 and the sockets descriptor has
	been set to -1, nil + error message otherwise.

`sock, addr, port = socket:accept()`
:	accept a new connection on a socket

	Returns a new socket with the accepted connection and ip address and
	port of the remote end on success, false if no connection is available
	to accept, or nil+error message on error. If called with a unix domain
	socket, addr and port will be nil on success.

	This only works on tcp type sockets.

`string = socket:recv( [size] )`
:	reads data from a socket

	** Arguments **

	* `size`: (optional) the length of the buffer to use for reading,
	  defaults to some internal value

	Returns a string containing the data read, false if no data was available
	to read, nil if the remote end closed the connection (tcp connections
	only), or nil+error message on error.

	This should only be used for tcp type sockets, or for udp type sockets
	that have been created by lsocket.connect(). For udp type sockets, that
	have been created with lsocket.bind(), see socket:recvfrom()

`string, address, port = socket:recvfrom( [size] )`
:	reads data from a socket

	** Arguments **

	* `size`: (optional) the length of the buffer to use for reading,
	  defaults to some internal value

	Returns a string containing the data read, and the ip address and port
	number of the remote end of the connection, false if no data was
	available to read, or nil+error message on error.

	This should only be used for udp type sockets. For tcp type
	sockets, see socket:recv()

`nbytes = socket:send(string)`
:	writes data to a socket

	** Arguments **

	* `string`: data to write to the socket

	Returns the number of bytes written, or false if the socket was not
	ready to accept data, or nil+error message on error.

	This should only be used for tcp type sockets, or for udp type
	sockets that have been created by lsocket.connect(). For udp
	type sockets, that have been created with lsocket.bind(), see
	socket:sendto()

`nbytes = socket:sendto(string, address, port)`
:	writes data to a socket

	** Arguments **

	* `string`: data to write to the socket
	* `address`: ip address of remote end of socket to send data to
	* `port`: port number of remote end of socket to send data to

	Returns the number of bytes written, or false if the socket was not
	ready to accept data, or nil+error message on error.

	This should only be used for udp type sockets. For tcp type
	sockets, see socket:send()

`ok = socket:close()`
:	closes a socket

	Returns true if closing the socket went ok, or nil+error message on
	error.

## Functions ##

`[read [, write]] = lsocket.select([read [, write]] , [timeout] )`
:	calls select() on up to 2 tables of sockets, has timeout

	** Arguments **

	* `read`: (opt) table of sockets to wait on for reading
	* `write`: (opt) table of sockets to wait on for writing
	* `timeout`: (opt) timeout in seconds (millisecond resolution),
	  defaults to infinite.

	Either only the read socket table or values for both socket
	tables must be given. That means, if you want to wait on sockets
	for writing, you will also have to pass a table or nil for
	reading. The timeout value can be specified without passing any
	tables before it, so that lsocket.select(timeout) can be used as
	a millisecond precision sleep function. Closed sockets are
	ignored by lsocket.select(), and it is an error to call
	lsocket.select() without any open sockets and timeout.

	Returns either as many tables as were passed as arguments, each one
	filled with the sockets that became ready from the select, false if a
	timeout occurred before any socket became ready, or nil+error message
	on error.

	Note: if you pass nil for the read sockets and some table for the write
	sockets, when a socket you wait on becomes ready, an empty table is
	returned as first return value.

`tbl = lsocket.resolve(name)`
:	attempts a name resolution of its argument.

	** Arguments **

	* `name`: hostname to find ip address(es) for

	Returns a table of ip addresses that the hostname resolves to. For each
	address, a record (subtable) is found in the result table with the fields

	* `family`: IP protocol family, "inet" or "inet6"
	* `addr`: the ip address

	On error, returns nil + error message.

	Note: this function does block!

`tbl = lsocket.getinterfaces()`
:	enumerate interfaces and their addresses
	
	Returns a list containing information about all available interfaces.
	For each interface, one or more records (subtables) are found in the
	result table with the fields

	* `name`: interface name
	* `family`: IP protocol family, "inet" or "inet6"
	* `addr`: the ip address
	* `mask`: the netmask

	On error, returns nil + error message.

## Constants ##

`lsocket.INADDR_ANY`
:	IPv4 "any" address, i.e. what you bind to if you intend to accept
	connections on all addresses your computer has.

`lsocket.IN6ADDR_ANY`
:	IPv6 "any" address, see lsocket.INADDR_ANY.

`lsocket._VERSION`
:	Version number of the lsocket module.

### Examples ###

There are a few examples in the samples folder, including a server and
client for tcp, udp and multicast. For all of those examples, if you
start them without command line arguments, they work with IPv4, if
start them with the argument 6, they work with IPv6.

