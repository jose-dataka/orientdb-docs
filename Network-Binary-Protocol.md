# Binary Protocol

Current protocol version for 2.0-SNAPSHOT: **28**. Look at [compatibility](#Compatibility) for retro-compatibility.

# Table of content
- [Introduction](#introduction)
	- [Connection](#connection)
	- [Getting started](#getting-started)
	- [Session](#session)
- [Enable debug messages on protocol](#enable-debug-messages-on-protocol)
- [Exchange](#exchange)
- [Network message format](#network-message-format)
- [Supported types](#supported-types)
- [Record format](#record-format)
- [Request](#request)
	- [Operation types](#operation-types)
- [Response](#response)
	- [Statuses](#statuses)
	- [Errors](#errors)
- [Operations](#operations)
	- [REQUEST_SHUTDOWN](#request_shutdown)
	- [REQUEST_CONNECT](#request_connect)
	- [REQUEST_DB_OPEN](#request_db_open)
	- [REQUEST_DB_CREATE](#request_db_create)
	- [REQUEST_DB_CLOSE](#request_db_close)
	- [REQUEST_DB_EXIST](#request_db_exist)
	- [REQUEST_DB_RELOAD](#request_db_reload)
	- [REQUEST_DB_DROP](#request_db_drop)
	- [REQUEST_DB_SIZE](#request_db_size)
	- [REQUEST_DB_COUNTRECORDS](#request_db_countrecords)
	- [REQUEST_DATACLUSTER_ADD](#request_datacluster_add)
	- [REQUEST_DATACLUSTER_DROP](#request_datacluster_drop)
	- [REQUEST_DATACLUSTER_COUNT](#request_datacluster_count)
		- [Example](#example)
	- [REQUEST_DATACLUSTER_DATARANGE](#request_datacluster_datarange)
		- [Example](#example)
	- [REQUEST_RECORD_LOAD](#request_record_load)
	- [REQUEST_RECORD_CREATE](#request_record_create)
	- [REQUEST_RECORD_UPDATE](#request_record_update)
	- [REQUEST_RECORD_DELETE](#request_record_delete)
	- [REQUEST_COMMAND](#request_command)
		- [SQL command payload](#sql-command-payload)
		- [SQL Script command payload](#sql-script-command-payload)
	- [REQUEST_TX_COMMIT](#request_tx_commit)
- [Special use of LINKSET types](#special-use-of-linkset-types)
	- [Tree node binary structure](#tree-node-binary-structure)
- [History](#history)
	- [Version 24](#version-24)
	- [Version 23](#version-23)
	- [Version 22](#version-22)
	- [Version 21](#version-21)
	- [Version 20](#version-20)
	- [Version 19](#version-19)
	- [Version 18](#version-18)
	- [Version 17](#version-17)
	- [Version 16](#version-16)
	- [Version 15](#version-15)
	- [Version 14](#version-14)
	- [Version 13](#version-13)
	- [Version 12](#version-12)
	- [Version 11](#version-11)
- [Compatibility](#compatibility)

# Introduction

The OrientDB binary protocol is the fastest way to interface a client application to an OrientDB Server instance. The aim of this page is to provide a starting point from which to build a language binding, maintaining high-performance.

If you'd like to develop a new binding, please take a look to the available ones before starting a new project from scratch: [Existent Drivers](Programming-Language-Bindings.md).

Also, check the available [REST implementations](OrientDB-REST.md).

Before starting, please note that:
- **[Record](Concepts.md#wiki-record)** is an abstraction of **[Document](Concepts.md#wiki-document)**. However, keep in mind that in OrientDB you can handle structures at a lower level than Documents. These include positional records, raw strings, raw bytes, etc.

For more in-depth information please look at the Java classes:
- Client side: [OStorageRemote.java](https://github.com/nuvolabase/orientdb/tree/master/client/src/main/java/com/orientechnologies/orient/client/remote/OStorageRemote.java)
- Server side: [ONetworkProtocolBinary.java](https://github.com/nuvolabase/orientdb/tree/master/server/src/main/java/com/orientechnologies/orient/server/network/protocol/binary/ONetworkProtocolBinary.java)
- Protocol constants: [OChannelBinaryProtocol.java](https://github.com/nuvolabase/orientdb/tree/master/enterprise/src/main/java/com/orientechnologies/orient/enterprise/channel/binary/OChannelBinaryProtocol.java)

## Connection

*(Since 0.9.24-SNAPSHOT Nov 25th 2010)* Once connected, the server sends a short number (2 byte) containing the binary protocol number. The client should check that it supports that version of the protocol. Every time the protocol changes the version is incremented.

## Getting started

After the connection has been established, a client can <b>Connect</b> to the server or request the opening of a database <b>Database Open</b>. Currently, only TCP/IP raw sockets are supported. For this operation use socket APIs appropriate to the language you're using. After the <b>Connect</b> and <b>Database Open</b> all the client's requests are sent to the server until the client closes the socket. When the socket is closed, OrientDB Server instance frees resources the used for the connection.

The first operation following the socket-level connection must be one of:
- [Connect to the server](#connect) to work with the OrientDB Server instance
- [Open a database](#db_open) to open an existing database

In both cases a [Session-Id](#Session-Id) is sent back to the client. The server assigns a unique Session-Id to the client. This value must be used for all further operations against the server. You may open a database after connecting to the server, using the same Session-Id

## Session
The session managment is implemented in two different way, one stateful another stateless this is choosed in the open/connect operation with a flag, the stateful is based on a [Session-id](#session-id) the stateless is based on a [Token](#token)

## Session-Id

All the operations that follow the open/connect must contain, as the first parameter, the client **Session-Id** (as Integer, 4 bytes) and it will be sent back on completion of the request just after the result field.

*NOTE: In order to create a new server-side connection, the client must send a negative number into the open/connect calls.*

This **Session-Id** can be used into the client to keep track of the requests if it handles multiple session bound to the same connection. In this way the client can implement a sharing policy to save resources. This requires that the client implementation handle the response returned and dispatch it to the correct caller thread.

## Token

All the operation in a stateless session are based on the token, the token is a byte[] that contains all the information for the interaction with the server, the token is acquired at the mement of open or connect, and need to be resend for each request. the session id used in the stateful requests is still there and is used to associate the request to the response. in the response can be resend a token in case of expire renew.

# Enable debug messages on protocol

To make the development of a new client easier it's strongly suggested to activate debug mode on the binary channel. To activate this, edit the file orientdb-server-config.xml and configure the new parameter "network.binary.debug" on the "binary" or "distributed" listener. E.g.:

```
...
<listener protocol="distributed" port-range="2424-2430"
ip-address="127.0.0.1">
<parameters>
<parameter name="network.binary.debug" value="true" />
</parameters>
</listener>
...
```

In the log file (or the console if you have configured the orientdb-server-log.properties file)
all the packets received will be printed.

# Exchange

This is the typical exchange of messages between client and server sides:
```
+------+ +------+
|Client| |Server|
+------+ +------+
| TCP/IP Socket connection |
+-------------------------->|
| DB_OPEN |
+-------------------------->|
| RESPONSE (+ SESSION-ID) |
+<--------------------------+
... ...
| REQUEST (+ SESSION-ID) |
+-------------------------->|
| RESPONSE (+ SESSION-ID) |
+<--------------------------+
... ...
| DB_CLOSE (+ SESSION-ID) |
+-------------------------->|
| TCP/IP Socket close |
+-------------------------->|
```
# Network message format

In explaining the network messages these conventions will be used:
- fields are bracketed by parenthesis and contain the name and the type separated by ':'. E.g. <code>(length:int)</code>

# Supported types

The network protocol supports different types of information:
<table>
<tbody>
<tr><th>Type</th><th>Minimum length in bytes</th><th>Maximum length in bytes</th><th>Notes</th><th>Example</th></tr>
<tr><th>boolean</th><td>1</td><td>1</td><td>Single byte: 1 = true, 0 = false</td><td>1</td></tr>
<tr><th>byte</th><td>1</td><td>1</td><td>Single byte, used to store small numbers and booleans</td><td>1</td></tr>
<tr><th>short</th><td>2</td><td>2</td><td>Signed short type</td><td>01</td></tr>
<tr><th>int</th><td>4</td><td>4</td><td>Signed integer type</td><td>0001</td></tr>
<tr><th>long</th><td>8</td><td>8</td><td>Signed long type</td><td>00000001</td></tr>
<tr><th>bytes</th><td>4</td><td>N</td><td>Used for binary data. The format is <code>(length:int)[bytes)]((content:&lt;length&gt;.md)</code>. Send -1 as NULL</td><td><code>000511111</code></td></tr>
<tr><th>string</th><td>4</td><td>N</td><td>Used for text messages.The format is: <code>(length:int)[bytes)]((content:&lt;length&gt;.md)</code>. Send -1 as NULL</td><td><code>0005Hello</code></td></tr>
<tr><th>record</th><td>2</td><td>N</td><td>An entire record serialized. The format depends if a RID is passed or an entire record with its content. In case of null record then -2 as short is passed. In case of RID -3 is passes as short and then the RID: <code>(-3:short)(cluster-id:short)(cluster-position:long)</code>. In case of record: <code>(0:short)(record-type:byte)(cluster-id:short)(cluster-position:long)(record-version:int)(record-content:bytes)</code></td><td></td></tr>
<tr><th>strings</th><td>4</td><td>N</td><td>Used for multiple text messages. The format is: <code>(length:int)[(Nth-string:string)]</code></td><td><code>00020005Hello0007World!</code></td></tr>
</tbody>
</table>

# Record format
The record format is choose during the [CONNECT](#request_connect) or [DB_OPEN](#request_db_open) request, the formats available are:

[CSV](Record-CSV-Serialization.md) (serialization-impl value "ORecordDocument2csv")
[Binary](Record-Schemaless-Binary-Serialization.md) (serialization-impl value "ORecordSerializerBinary")

The CSV format is the default for all the versions 0.* and 1.* or for any client with Network Protocol Version < 22

# Request

Each request has own format depending of the operation requested. The operation requested is indicated in the first byte:
- *1 byte* for the operation. See [Operation types](#Operation_types) for the list
- **4 bytes** for the [Session-Id](#Session-Id) number as Integer
- **N bytes** optional token bytes only present if the REQUEST_CONNECT/REQUEST_DB_OPEN return a token.
- **N bytes** = message content based on the operation type

## Operation types

<table>
<tr><th>Command</th><th>Value as byte</th><th>Description</th><th>Async</th><th>Since</th></tr>

<tr><td colspan="5"><h3>Server <i>(CONNECT Operations)</i></h3></td></tr>
<tr><td><a href="#request_shutdown">REQUEST_SHUTDOWN</a></td><td>1</td><td>Shut down server.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_connect">REQUEST_CONNECT</a></td><td>2</td><td>Required initial operation</b> to access to server commands.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_db_open">REQUEST_DB_OPEN</a></td><td>3</td><td><b>Required initial operation</b> to access to the database.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_db_create">REQUEST_DB_CREATE</a></td><td>4</td><td>Add a new database.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_db_exist">REQUEST_DB_EXIST</a></td><td>6</td><td>Check if database exists.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_db_drop">REQUEST_DB_DROP</a></td><td>7</td><td>Delete database.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_CONFIG_GET</td><td>70</td><td>Get a configuration property.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_CONFIG_SET</td><td>71</td><td>Set a configuration property.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_CONFIG_LIST</td><td>72</td><td>Get a list of configuration properties.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_DB_LIST<td>74</td><td>Get a list of databases.</td><td>no</td><td>1.0rc6</td></tr>

<tr><td colspan="5"><h3>Database <i>(DB_OPEN Operations)</i></h3></td></tr>
<tr><td><a href="#request_db_close">REQUEST_DB_CLOSE</a></td><td>5</td><td>Close a database.</td><td>no</td><td></td></tr>

<tr><td><a href="#request_db_size">REQUEST_DB_SIZE</a></td><td>8</td><td>Get the size of a database (in bytes).</td><td>no</td><td>0.9.25</td></tr>
<tr><td><a href="#request_db_countrecords">REQUEST_DB_COUNTRECORDS</a></td><td>9</td><td>Get total number of records in a database.</td><td>no</td><td>0.9.25</td></tr>
<tr><td><a href="#request_datacluster_add">REQUEST_DATACLUSTER_ADD</a></td><td>10</td><td>Add a data cluster.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_datacluster_drop">REQUEST_DATACLUSTER_DROP</a><td>11</td><td>Delete a data cluster.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_datacluster_count">REQUEST_DATACLUSTER_COUNT</a></td><td>12</td><td>Get the total number of data clusters.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_datacluster_datarange">REQUEST_DATACLUSTER_DATARANGE</a></td><td>13</td><td>Get the data range of data clusters.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_DATACLUSTER_COPY</td><td>14</td><td>Copy a data cluster.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_DATACLUSTER_LH_CLUSTER_IS_USED</td><td>16</td><td></td><td>no</td><td>1.2.0</td></tr>
<tr><td>REQUEST_RECORD_METADATA</td><td>29</td><td>Get metadata from a record.</td><td>no</td><td>1.4.0</td></tr>
<tr><td><a href="#request_record_load">REQUEST_RECORD_LOAD</a></td><td>30</td><td>Load a record.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_record_create">REQUEST_RECORD_CREATE</a></td><td>31</td><td>Add a record.</td><td>yes</td><td></td></tr>
<tr><td><a href="#request_record_update">REQUEST_RECORD_UPDATE</a></td><td>32</td><td><Update a record./td><td>yes</td><td></td></tr>
<tr><td><a href="#request_record_delete">REQUEST_RECORD_DELETE</a></td><td>33</td><td>Delete a record.</td><td>yes</td><td></td></tr>
<tr><td>REQUEST_RECORD_COPY</td><td>34</td><td>Copy a record.</td><td>yes</td><td></td></tr>
<tr><td>REQUEST_RECORD_CLEAN_OUT</td><td>38</td><td>Clean out record.</td><td>yes</td><td>1.3.0</td></tr>
<tr><td>REQUEST_POSITIONS_FLOOR</td><td>39</td><td>Get the last record.</td><td>yes</td><td>1.3.0</td></tr>
<tr><td>REQUEST_COUNT <i>(DEPRECATED)</i></td><td>40</td><td>See REQUEST_DATACLUSTER_COUNT</td><td>no</td><td></td></tr>
<tr><td><a href="#request_command">REQUEST_COMMAND</a></td><td>41</td><td>Execute a command.</td><td>no</td><td></td></tr>
<tr><td>REQUEST_POSITIONS_CEILING</td><td>42</td><td>Get the first record.</td><td>no</td><td>1.3.0</td></tr>
<tr><td><a href="#request_tx_commit">REQUEST_TX_COMMIT</a></td><td>60</td><td>Commit transaction.</td><td>no</td><td></td></tr>
<tr><td><a href="#request_db_reload">REQUEST_DB_RELOAD</a></td><td>73</td><td>Reload database.</td><td>no</td><td>1.0rc4</td></tr>
<tr><td>REQUEST_PUSH_RECORD<td>79</td><td></td><td>no</td><td>1.0rc6</td></tr>
<tr><td>REQUEST_PUSH_DISTRIB_CONFIG<td>80</td><td></td><td>no</td><td>1.0rc6</td></tr>
<tr><td>REQUEST_DB_COPY<td>90</td><td></td><td>no</td><td>1.0rc8</td></tr>
<tr><td>REQUEST_REPLICATION<td>91</td><td></td><td>no</td><td>1.0</td></tr>
<tr><td>REQUEST_CLUSTER<td>92</td><td></td><td>no</td><td>1.0</td></tr>
<tr><td>REQUEST_DB_TRANSFER<td>93</td><td></td><td>no</td><td>1.0.2</td></tr>
<tr><td>REQUEST_DB_FREEZE<td>94</td><td></td><td>no</td><td>1.1.0</td></tr>
<tr><td>REQUEST_DB_RELEASE<td>95</td><td></td><td>no</td><td>1.1.0</td></tr>
<tr><td>REQUEST_DATACLUSTER_FREEZE<td>96</td><td></td><td>no</td><td></td></tr>
<tr><td>REQUEST_DATACLUSTER_RELEASE<td>97</td><td></td><td>no</td><td></td></tr>
<tr><td>REQUEST_CREATE_SBTREE_BONSAI<td>110</td><td>Creates an sb-tree bonsai on the remote server</td><td>no</td><td>1.7rc1</td></tr>
<tr><td>REQUEST_SBTREE_BONSAI_GET<td>111</td><td>Get value by key from sb-tree bonsai</td><td>no</td><td>1.7rc1</td></tr>
<tr><td>REQUEST_SBTREE_BONSAI_FIRST_KEY<td>112</td><td>Get first key from sb-tree bonsai</td><td>no</td><td>1.7rc1</td></tr>
<tr><td>REQUEST_SBTREE_BONSAI_GET_ENTRIES_MAJOR<td>113</td><td>Gets the portion of entries major than specified one. If returns 0 entries than the specified entrie is the largest</td><td>no</td><td>1.7rc1</td></tr>
<tr><td>REQUEST_RIDBAG_GET_SIZE<td>114</td><td>Rid-bag specific operation. Send but does not save changes of rid bag. Retrieves computed size of rid bag.</td><td>no</td><td>1.7rc1</td></tr>
</table>

# Response

Every request has a response unless the command supports the asynchronous mode (look at the table above).
- **1 byte**: Success status of the request if succeeded or failed (0=OK, 1=ERROR)
- **4 bytes**: [Session-Id](#Session-Id) (Integer)
- **N bytes** optional token, is only present for token based session (REQUEST_CONNECT/REQUEST_DB_OPEN return a token) and is usually empty(N=0) is only filled up by the server when renew of an expiring token is required.
- **N bytes**: Message content depending on the operation requested

## Statuses

Every time the client sends a request, and the command is not in asynchronous mode (look at the table above), client must read the one-byte response status that indicates OK or ERROR. The rest of response bytes depends on this first byte.
```
* OK = 0;
* ERROR = 1;
```


**OK response bytes are depends for every request type. ERROR response bytes sequence described below.**

## Errors

The format is: `[(1)(exception-class:string)(exception-message:string)]*(0)(serialized-exception:bytes)`

The pairs exception-class and exception-message continue while the following byte is 1. A 0 in this position indicates that no more data follows.

E.g. (parentheses are used here just to separate fields to make this easier to read: they are not present in the server response):
```
(1)(com.orientechnologies.orient.core.exception.OStorageException)(Can't open the storage 'demo')(0)
```
Example of 2 depth-levels exception:
```
(1)(com.orientechnologies.orient.core.exception.OStorageException)(Can't open the storage 'demo')(1)(com.orientechnologies.orient.core.exception.OStorageException)(File not found)(0)
```

Since 1.6.1 we also send serialized version of exception thrown on server side. This allows to preserve full
stack trace of server exception on client side but this feature can be used by Java clients only.

# Operations

This section explains the *request* and *response* messages of all suported operations.

## REQUEST_SHUTDOWN

Shut down the server. Requires "shutdown" permission to be set in *orientdb-server-config.xml* file.

```
Request: (user-name:string)(user-password:string)
Response: empty
```
Typically the credentials are those of the OrientDB server administrator. This is not the same as the *admin* user for individual databases.
## REQUEST_CONNECT

This is the first operation requested by the client when it needs to work with the server instance. It returns the session id of the client.

```
Request: (driver-name:string)(driver-version:string)(protocol-version:short)(client-id:string)(serialization-impl:string)(token-session:boolean)(user-name:string)(user-password:string)
Response: (session-id:int)(token:bytes)
```
Where:  
request content:  
- client's **driver-name** as string. Example: "OrientDB Java client"
- client's **driver-version** as string. Example: "1.0rc8-SNAPSHOT"
- client's **protocol-version** as short. Example: 7
- client's **client-id** as string. Can be null for clients. In clustered configuration is the distributed node ID as TCP host+port. Example: "10.10.10.10:2480"
- client's **serialization-impl** the [serialization format](#record-format) required by the client.
- **token-session** as boolean, true if the client want to use a token based session otherwise false
- **user-name** as string. Example: "root"
- **user-password** as string. Example: "kdsjkkasjad"
Typically the credentials are those of the OrientDB server administrator. This is not the same as the *admin* user for individual databases. It returns the [Session-Id](#Session-Id) to being reused for all the next calls.  

response content:  
- **session-id** the new session id or a match id in case of token auth  
- **token:bytes** the token bytes or empty(size = 0) if the client send token-session=false or the server not support the token based session  


## REQUEST_DB_OPEN

This is the first operation the client should call. It opens a database on the remote OrientDB Server. Returns the [Session-Id](#Session-Id) to being reused for all the next calls and the list of configured [clusters](Concepts.md#wiki-Cluster).

```
Request: (driver-name:string)(driver-version:string)(protocol-version:short)(client-id:string)(serialization-impl:string)(token-session:boolean)(database-name:string)(database-type:string)(user-name:string)(user-password:string)
Response: (session-id:int)(token:bytes)(num-of-clusters:short)[(cluster-name:string)(cluster-id:short)](cluster-config:bytes.md)(orientdb-release:string)
```
Where:
request detail:
- client's **driver-name** as string. Example: "OrientDB Java client"
- client's **driver-version** as string. Example: "1.0rc8-SNAPSHOT"
- client's **protocol-version** as short. Example: 7
- client's **client-id** as string. Can be null for clients. In clustered configuration is the distributed node ID as TCP host+port. Example: "10.10.10.10:2480"
- client's *serialization-impl* the [serialization format](#record-format) required by the client.
- **token-session** as boolean, true if the client want to use a token based session otherwise false
- **database-name** as string. Example: "demo"
- **database-type** as string, can be 'document' or 'graph' (since version 8). Example: "document"
- **user-name** as string. Example: "admin"
- **user-password** as string. Example: "admin"
- **cluster-config** is always null unless you're running in a server clustered configuration.
- **orientdb-release** as string. Contains version of OrientDB release deployed on server and optionally build number. Example: "1.4.0-SNAPSHOT (build 13)"

response detail :   
- **session-id** the new session id or a match id in case of token auth  
- **token:bytes** the token bytes or empty(size = 0) if the client send token-session=false or the server not support the token based session  
- **num-of-clusters:short** the size of cluster definition array composed by **cluster-name:string** and **cluster-id:short** and **cluster-config**  
-  **orientdb-release** the decription of the token release  

## REQUEST_DB_CREATE

Creates a database in the remote OrientDB server instance

```
Request: (database-name:string)(database-type:string)(storage-type:string)
Response: empty
```
Where:
- **database-name** as string. Example: "demo"
- **database-type** as string, can be 'document' or 'graph' (since version 8). Example: "document"
- **storage-type** can be one of the [supported types](Concepts.md#wiki-Database_URL):
- plocal, as a persistent database
- memory, as a volatile database

NB. It doesn't make sense to use "remote" in this context

## REQUEST_DB_CLOSE

Closes the database and the network connection to the OrientDB Server instance. No return is expected. The socket is also closed.

```
Request: empty
Response: no response, the socket is just closed at server side
```

## REQUEST_DB_EXIST

Asks if a database exists in the OrientDB Server instance. It returns true (non-zero) or false (zero).

```xml
Request: (database-name:string) <-- before 1.0rc1 this was empty (server-storage-type:string - since 1.5-snapshot)
Response: (result:byte)
```

Where:
- **server-storage-type** can be one of the [supported types](Concepts.md#wiki-Database_URL):
- plocal as a persistent database
- memory, as a volatile database


## REQUEST_DB_RELOAD

Reloads database information. Available since 1.0rc4.

```
Request: empty
Response:(num-of-clusters:short)[(cluster-name:string)(cluster-id:short)]
```

## REQUEST_DB_DROP

Removes a database from the OrientDB Server instance. It returns nothing if the database has been deleted or throws a OStorageException if the database doesn't exists.

```
Request: (database-name:string)(server-storage-type:string - since 1.5-snapshot)
Response: empty
```

Where:
- **server-storage-type** can be one of the [supported types](Concepts.md#wiki-Database_URL):
- plocal as a persistent database
- memory, as a volatile database

## REQUEST_DB_SIZE

Asks for the size of a database in the OrientDB Server instance.

```
Request: empty
Response: (size:long)
```

## REQUEST_DB_COUNTRECORDS

Asks for the number of records in a database in the OrientDB Server instance.

```
Request: empty
Response: (count:long)
```

## REQUEST_DATACLUSTER_ADD

Add a new data cluster.
```
Request: (name:string)(cluster-id:short - since 1.6 snapshot)
Response: (new-cluster:short)
```
Where: type is one of "PHYSICAL" or "MEMORY".
If cluster-id is -1 (recommended value) new cluster id will be generated.

## REQUEST_DATACLUSTER_DROP

Remove a cluster.
```
Request: (cluster-number:short)
Response: (delete-on-clientside:byte)
```

Where:
- **delete-on-clientside** can be 1 if the cluster has been successfully removed and the client has to remove too, otherwise 0

## REQUEST_DATACLUSTER_COUNT

Returns the number of records in one or more clusters.
```
Request: (cluster-count:short)(cluster-number:short)*(count-tombstones:byte)
Response: (records-in-clusters:long)
```
Where:
- **cluster-count** the number of requested clusters
- **cluster-number** the cluster id of each single cluster
- **count-tombstones** the flag which indicates whether deleted records should be taken in account. It is applicable for autosharded storage only, otherwise it is ignored.
- **records-in-clusters** is the total number of records found in the requested clusters

### Example

Request the record count for clusters 5, 6 and 7. Note the "03" at the beginning to tell you're passing 3 cluster ids (as short each). 1,000 as long (8 bytes) is the answer.
```
Request: 03050607
Response: 00001000
```

## REQUEST_DATACLUSTER_DATARANGE

Returns the range of record ids for a cluster.
```
Request: (cluster-number:short)
Response: (begin:long)(end:long)
```
### Example

Request the range for cluster 7. The range 0-1,000 is returned in the response as 2 longs (8 bytes each).
```
Request: 07
Response: 0000000000001000
```
## REQUEST_RECORD_LOAD

Load a record by [RecordID](Concepts.md#RecordID), according to a [fetch plan](Fetching-Strategies.md)

```
Request: (cluster-id:short)(cluster-position:long)(fetch-plan:string)(ignore-cache:byte)(load-tombstones:byte)
Response: [(payload-status:byte)[(record-type:byte)(record-version:int)(record-content:bytes)]*]+
```
Where:
- **fetch-plan**, the [fetch plan](Fetching-Strategies.md) to use or an empty string
- **ignore-cache**, tells if the cache must be ignored: 1 = ignore the cache, 0 = not ignore. since protocol v.9 (introduced in release 1.0rc9)
- **load-tombstones**, the flag which indicates whether information about deleted record should be loaded. The flag is applied only to autosharded storage and ignored otherwise.
- **payload-status** can be:
- 0: no records remain to be fetched
- 1: a record is returned as resultset
- 2: a record is returned as pre-fetched to be loaded in client's cache only. It's not part of the result set but the client knows that it's available for later access. This value is not currently used.
- **record-type** is
- 'b': raw bytes
- 'f': flat data
- 'd': document

## REQUEST_RECORD_CREATE

Create a new record. Returns the position in the cluster of the new record. New records can have version > 0 (since v1.0) in case the RID has been recycled.

```
Request: (cluster-id:short)(record-content:bytes)(record-type:byte)(mode:byte)
Response: (cluster-id:short)(cluster-position:long)(record-version:int)(count-of-collection-changes)[(uuid-most-sig-bits:long)(uuid-least-sig-bits:long)(updated-file-id:long)(updated-page-index:long)(updated-page-offset:int)]*
```
Where:
- **datasegment-id** the segment id to store the data (since version 10 - 1.0-SNAPSHOT). -1 Means default one. Removed since 2.0
- **record-type** is:
- 'b': raw bytes
- 'f': flat data
- 'd': document

and **mode** is:
- 0 = synchronous (default mode waits for the answer)
- 1 = asynchronous (don't need an answer)

The last part of response is referred to [RidBag](RidBag.md) management. Take a look at [the main page](RidBag.md) for more details.

## REQUEST_RECORD_UPDATE

Update a record. Returns the new record's version.
```
Request: (cluster-id:short)(cluster-position:long)(update-content:boolean)(record-content:bytes)(record-version:int)(record-type:byte)(mode:byte)
Response: (record-version:int)(count-of-collection-changes)[(uuid-most-sig-bits:long)(uuid-least-sig-bits:long)(updated-file-id:long)(updated-page-index:long)(updated-page-offset:int)]*
```
Where **record-type** is:
- 'b': raw bytes
- 'f': flat data
- 'd': document

and **record-version** **policy** is:
- '-1': Document update, version increment, no version control.
- '-2': Document update, no version control nor increment.
- '-3': Used internal in transaction rollback (version decrement).
- '>-1': Standard document update (version control).

and **mode** is:
- 0 = synchronous (default mode waits for the answer)
- 1 = asynchronous (don't need an answer)

and **update-content** is:
- true - content of record has been changed and content should be updated in storage
- false - the record was modified but its own content has not been changed. So related collections (e.g. rig-bags) have to be updated, but record version and content should not be.

The last part of response is referred to [RidBag](RidBag.md) management. Take a look at [the main page](RidBag.md) for more details.

## REQUEST_RECORD_DELETE

Delete a record by its [RecordID](Concepts.md#RecordID). During the optimistic transaction the record will be deleted only if the versions match. Returns true if has been deleted otherwise false.

```
Request: (cluster-id:short)(cluster-position:long)(record-version:int)(mode:byte)
Response: (payload-status:byte)
```

Where:
- **mode** is:
- 0 = synchronous (default mode waits for the answer)
- 1 = asynchronous (don't need an answer)
- **payload-status** returns 1 if the record has been deleted, otherwise 0. If the record didn't exist 0 is returned.

## REQUEST_COMMAND

Executes remote commands:

```
Request: (mode:byte)(command-payload-length:int)(class-name:string)(command-payload)
Response:
- synchronous commands: [(synch-result-type:byte)[(synch-result-content:?)]]+
- asynchronous commands: [(asynch-result-type:byte)[(asynch-result-content:?)]*](pre-fetched-record-size.md)[(pre-fetched-record)]*+
```

Where the request:
- **mode** can be 'a' for asynchronous mode and 's' for synchronous mode
- **command-payload-length** is the length of the class-name field plus the command-payload field
- **class-name** is the class name of the command implementation. There are short form for the most common commands:
 - **q** stands for query as idempotent command. It's like passing <code>com.orientechnologies.orient.core.sql.query.OSQLSynchQuery</code>
 - **c** stands for command as non-idempotent command (insert, update, etc). It's like passing <code>com.orientechnologies.orient.core.sql.OCommandSQL</code>
 - **s** stands for script. It's like passing <code>com.orientechnologies.orient.core.command.script.OCommandScript</code>. Script commands by using any supported server-side scripting like [Javascript command](Javascript-Command.md). Since v1.0.
 - **any other values** is the class name. The command will be created via reflection using the default constructor and invoking the <code>fromStream()</code> method against it
- **command-payload** is the command's serialized payload (see [Network-Binary-Protocol-Commands](Network-Binary-Protocol-Commands.md))

Response is different for synchronous and asynchronous request:
- **synchronous**:
- **synch-result-type** can be:
 - 'n', means null result
 - 'r', means single record returned
 - 'l', collection of records. The format is:
  - an integer to indicate the collection size
  - all the records one by one
 - 'a', serialized result, a byte[] is sent
- **synch-result-content**, can only be a record
- **pre-fetched-record-size**, as the number of pre-fetched records not directly part of the result set but joined to it by fetching
- **pre-fetched-record** as the pre-fetched record content
- **asynchronous**:
- **asynch-result-type** can be:
 - 0: no records remain to be fetched
 - 1: a record is returned as a resultset
 - 2: a record is returned as pre-fetched to be loaded in client's cache only. It's not part of the result set but the client knows that it's available for later access
- **asynch-result-content**, can only be a record


## REQUEST_TX_COMMIT

Commits a transaction. This operation flushes all the pending changes to the server side.
```
Request: (tx-id:int)(using-tx-log:byte)(tx-entry)*(0-byte indicating end-of-records)

 tx-entry: (operation-type:byte)(cluster-id:short)(cluster-position:long)(record-type:byte)(entry-content)
  entry-content for CREATE: (record-content:bytes)
  entry-content for UPDATE: (version:record-version)(content-changed:boolean)(record-content:bytes)
  entry-content for DELETE: (version:record-version)

Response: (created-record-count:int)[(client-specified-cluster-id:short)(client-specified-cluster-position:long)(created-cluster-id:short)(created-cluster-position:long)]*(updated-record-count:int)[(updated-cluster-id:short)(updated-cluster-position:long)(new-record-version:int)]*(count-of-collection-changes:int)[(uuid-most-sig-bits:long)(uuid-least-sig-bits:long)(updated-file-id:long)(updated-page-index:long)(updated-page-offset:int)]*
```

Where:
- **tx-id** is the Transaction's Id
- **using-tx-log** tells if the server must use the Transaction Log to recover the transaction. 1 = true, 0 = false. Use always 1 (true) by default to assure consistency. _NOTE: Disabling the log could speed up execution of transaction, but can't be rollbacked in case of error. This could bring also at inconsistency in indexes as well, because in case of duplicated keys the rollback is not called to restore the index status._
- **operation-type** can be:
 - 1, for **UPDATES**
 - 2, for **DELETES**
 - 3, for **CREATIONS**
- **record-content** depends on the operation type:
 - For **UPDATED** (1): <code>(original-record-version:int)(record-content:bytes)</code>
 - For **DELETED** (2): <code>(original-record-version:int)</code>
 - For **CREATED** (3): <code>(record-content:bytes)</code>

This response contains two parts: a map of 'temporary' client-generated record ids to 'real' server-provided record ids for each CREATED record, and a map of UPDATED record ids to update record-versions.

Look at [Optimistic Transaction](Transactions.md#wiki-Optimistic_Transaction) to know how temporary [RecordID](Concepts.md#RecordID)s are managed.

The last part or response is referred to [RidBag](RidBag.md) management. Take a look at [the main page](RidBag.md) for more details.

### REQUEST_CREATE_SBTREE_BONSAI
```
Request: (clusterId:int)
Response: (collectionPointer)
```
See: [serialization of collection pointer](RidBag.md#serialization-of-collection-pointer)

Creates an sb-tree bonsai on the remote server.

### REQUEST_SBTREE_BONSAI_GET
```
Request: (collectionPointer)(key:binary)
Response: (valueSerializerId:byte)(value:binary)
```
See: [serialization of collection pointer](RidBag.md#serialization-of-collection-pointer)

Get value by key from sb-tree bonsai.

Key and value are serialized according to format of tree serializer.
If the operation is used by RidBag key is always a RID and value can be null or integer.


### REQUEST_SBTREE_BONSAI_FIRST_KEY
```
Request: (collectionPointer)
Response: (keySerializerId:byte)(key:binary)
```
See: [serialization of collection pointer](RidBag.md#serialization-of-collection-pointer)

Get first key from sb-tree bonsai. Null if tree is empty.

Key are serialized according to format of tree serializer.
If the operation is used by RidBag key is null or RID.

### REQUEST_SBTREE_BONSAI_GET_ENTRIES_MAJOR
```
Request: (collectionPointer)(key:binary)(inclusive:boolean)(pageSize:int)
Response: (count:int)[(key:binary)(value:binary)]*
```
See: [serialization of collection pointer](RidBag.md#serialization-of-collection-pointer)

Gets the portion of entries major than specified one. If returns 0 entries than the specified entry is the largest.

Keys and values are serialized according to format of tree serializer.
If the operation is used by RidBag key is always a RID and value is integer.

Default pageSize is 128.

### REQUEST_RIDBAG_GET_SIZE
```
Request: (collectionPointer)(collectionChanges)
Response: (size:int)
```
See: [serialization of collection pointer](RidBag.md#serialization-of-collection-pointer), [serialization of collection changes](RidBag.md#serialization-of-rid-bag-changes)

Rid-bag specific operation. Send but does not save changes of rid bag. Retrieves computed size of rid bag.

# Special use of LINKSET types
> NOTE. Since 1.7rc1 this feature is deprecated. Usage of RidBag is preferable.

Starting from 1.0rc8-SNAPSHOT OrientDB can transform collections of links from the classic mode:
```
[#10:3,#10:4,#10:5]
```
to:
```
(ORIDs@pageSize:16,root:#2:6)
```

For more information look at the announcement of this new feature: https://groups.google.com/d/topic/orient-database/QF52JEwCuTM/discussion

In practice to optimize cases with many relationships/edges the collection is transformed in a mvrb-tree. This is because the embedded object. In that case the important thing is the link to the root node of the balanced tree.

You can disable this behaviour by setting

*mvrbtree.ridBinaryThreshold* = -1

Where *mvrbtree.ridBinaryThreshold* is the threshold where OrientDB will use the tree instead of plain collection (as before). -1 means "hey, never use the new mode but leave all as before".

## Tree node binary structure

To improve performance this structure is managed in binary form. Below how is made:
```
+-----------+-----------+--------+------------+----------+-----------+---------------------+
| TREE SIZE | NODE SIZE | COLOR .| PARENT RID | LEFT RID | RIGHT RID | RID LIST .......... |
+-----------+-----------+--------+------------+----------+-----------+---------------------+
| 4 bytes . | 4 bytes . | 1 byte | 10 bytes ..| 10 bytes | 10 bytes .| 10 * MAX_SIZE bytes |
+-----------+-----------+--------+------------+----------+-----------+---------------------+
= 39 bytes + 10 * PAGE-SIZE bytes
```

Where:
- *TREE SIZE* as signed integer (4 bytes) containing the size of the tree. Only the root node has this value updated, so to know the size of the collection you need to load the root node and get this field. other nodes can contain not updated values because upon rotation of pieces of the tree (made during tree rebalancing) the root can change and the old root will have the "old" size as dirty.
- *NODE SIZE* as signed integer (4 bytes) containing number of entries in this node. It's always <= to the page-size defined at the tree level and equals for all the nodes. By default page-size is 16 items
- *COLOR* as 1 byte containing 1=Black, 0=Red. To know more about the meaning of this look at [Red-Black Trees](http://en.wikipedia.org/wiki/Red%E2%80%93black_tree)
- **PARENT RID** as [RID](Concepts.md#RecordID) (10 bytes) of the parent node record
- **LEFT RID** as [RID](Concepts.md#RecordID) (10 bytes) of the left node record
- **RIGHT RID** as [RID](Concepts.md#RecordID) (10 bytes) of the right node record
- **RID LIST** as the list of [RIDs](Concepts.md#RecordID) containing the references to the records. This is pre-allocated to the configured page-size. Since each [RID](Concepts.md#RecordID) takes 10 bytes, a page-size of 16 means 16 x 10bytes = 160bytes

The size of the tree-node on disk (and memory) is fixed to avoid fragmentation. To compute it: 39 bytes + 10 * PAGE-SIZE bytes. For a page-size = 16 you'll have 39 + 160 = 199 bytes.

# History
## Version 28
Since version 28 the [REQUEST_RECORD_LOAD](#request_record_load) response order is changed from:  
`[(payload-status:byte)[(record-content:bytes)(record-version:int)(record-type:byte)]*]+`
to:  
`[(payload-status:byte)[(record-type:byte)(record-version:int)(record-content:bytes)]*]+`

## Version 27

Since version 27 is introduced an extension to allow use a token based session, if this modality is enabled a few things change in the modality the protocol works.

* in the first negotiation the client should ask for a token based authentication using the token-auth flag
* the server will reply with a token or an empty byte array that means that it not support token based session and is using a old style session.
* if the server don't send back the token the client can fail or drop back the the old modality.
* for each request the client should send the token and the sessionId
* the sessionId is needed only for match a response to a request
* if used the token the connections can be shared between users and db of the same server, not needed to have connection associated to db and user.

protocol methods changed:  

REQUEST_DB_OPEN  
* request add token session flag
* response add of the token  

REQUEST_CONNECT  
* request add token session flag
* response add of the token 

## Version 26
added cluster-id in the REQUEST_CREATE_RECORD response.

## Version 25
Reviewd serialization of index changes in the REQUEST_TX_COMMIT for detais [#2676](https://github.com/orientechnologies/orientdb/issues/2676)
Removed double serialization of commands parameters, now the parameters are directly serialized in a document see [Network Binary Protocol Commands](Network-Binary-Protocol-Commands.md) and [#2301](https://github.com/orientechnologies/orientdb/issues/2301)

## Version 24
 - cluster-type and cluster-dataSegmentId parameters were removed from response for REQUEST_DB_OPEN, REQUEST_DB_RELOAD requests.
 - datasegment-id parameter was removed from REQUEST_RECORD_CREATE request.
 - type, location and datasegment-name parameters were removed from REQUEST_DATACLUSTER_ADD request.
 - REQUEST_DATASEGMENT_ADD  request was removed.
 - REQUEST_DATASEGMENT_DROP request was removed.

## Version 23
 - Add support of `updateContent` flag to UPDATE_RECORD and COMMIT

## Version 22
 - REQUEST_CONNECT and REQUEST_OPEN now send the document serialization format that the client require

## Version 21
- REQUEST_SBTREE_BONSAI_GET_ENTRIES_MAJOR (which is used to iterate through SBTree) now gets "pageSize" as int as last argument. Version 20 had a fixed pageSize=5. The new version provides configurable pageSize by client. Default pageSize value for protocol=20 has been changed to 128.

## Version 20
- Rid bag commands were introduced.
- Save/commit was adapted to support client notifications about changes of collection pointers.

## Version 19
- Serialized version of server exception is sent to the client.

## Version 18
- Ability to set cluster id during cluster creation was added.

## Version 17
- Synchronous commands can send fetched records like asynchronous one.

## Version 16
- Storage type is required for REQUEST_DB_FREEZE, REQUEST_DB_RELEASE, REQUEST_DB_DROP, REQUEST_DB_EXIST commands.
- This is required to support plocal storage.

## Version 15
- SET types are stored in different way then LIST. Before rel. 15 both were stored between squared braces [] while now SET are stored between <>

## Version 14
- DB_OPEN returns information about version of OrientDB deployed on server.

## Version 13
- To support upcoming auto-sharding support feature following changes were done
  - RECORD_LOAD flag to support ability to load tombstones was added.
  - DATACLUSTER_COUNT flag to support ability to count tombstones in cluster was added.

## Version 12
- DB_OPEN returns the dataSegmentId foreach cluster

## Version 11
- RECORD_CREATE always returns the record version. This was necessary because new records could have version > 0 to avoid MVCC problems on RID recycle

# Compatibility

Current release of OrientDB server supports older client versions.
- version 26: 100% compatible 2.0-SNAPSHOT
- version 25: 100% compatible 2.0-SNAPSHOT
- version 24: 100% compatible 2.0-SNAPSHOT
- version 23: 100% compatible 2.0-SNAPSHOT
- version 22: 100% compatible 2.0-SNAPSHOT
- version 22: 100% compatible 2.0-SNAPSHOT
- version 21: 100% compatible 1.7-SNAPSHOT
- version 20: 100% compatible 1.7rc1-SNAPSHOT
- version 19: 100% compatible 1.6.1-SNAPSHOT
- version 18: 100% compatible 1.6-SNAPSHOT
- version 17: 100% compatible. 1.5
- version 16: 100% compatible. 1.5-SNAPSHOT
- version 15: 100% compatible. 1.4-SNAPSHOT
- version 14: 100% compatible. 1.4-SNAPSHOT
- version 13: 100% compatible. 1.3-SNAPSHOT
- version 12: 100% compatible. 1.3-SNAPSHOT
- version 11: 100% compatible. 1.0-SNAPSHOT
- version 10: 100% compatible. 1.0rc9-SNAPSHOT
- version 9: 100% compatible. 1.0rc9-SNAPSHOT
- version 8: 100% compatible. 1.0rc9-SNAPSHOT
- version 7: 100% compatible. 1.0rc7-SNAPSHOT - 1.0rc8
- version 6: 100% compatible. Before 1.0rc7-SNAPSHOT
- < version 6: not compatible