---
title: "Remote Procedure Calls in Go"
date: 2021-09-27T13:44:57Z
tags : ['rpc', 'go', 'vagrant']
draft : true
---

## Introduction
These days most networked applications are designed a logical, self-contained, unifunctional services. Each service communicates with other services to perform some actions such as completing
user requests. For example, a notification service may contact a user management service to verify the authenticity of the user. Such inter-service communication requires communication protocal such as REST and GraphQL. In this post, I want to briefly introduce another method called Remote
Procedure Calls (RPC).

### Remote Procedure Calls (RPC)
RPC is a communication protocol that allows one service to call a function/method/procedure in another service over a network. Essentially, the client service encodes the function definition and potential arguments into some format and sends it over to the server service. The server decoded the message and executes the specified method (using the arguments) and returns a response to the client. There several frameworks that
simplify the definition and usage of RPCs. These include gRPC and Apache Thrift. In this post, I want to discuss a simple RPC setup using Go's RPC package.

### Go's RPC package
Per the documentation, the Go RPC packages "provides access to the exported methods of an object across a network or other I/O connection". The methods that are made available must satisfy the following criteria:

```text
- the method's type is exported.
- the method is exported.
- the method has two arguments, both exported (or builtin) types.
- the method's second argument is a pointer.
- the method has return type error.
```
Basically, a method whose signature matches the following template:
```
func (t *T) MethodName(argType T1, replyType *T2) error
```

To demonstrate how this will work, I'll simulate a simple environment with two servers, each running a simple process, communicating using RPC. The server will be created and networked using Vagrant (with Virtualbox).

### Vagrant Servers
The Vagrantfile looks like so:
```ruby
Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/focal64"

  config.vm.define "athena" do |athena|
        athena.vm.network "private_network", ip: "192.168.33.90"
        athena.vm.network "forwarded_port", guest: "7077", host: "7077"

        athena.vm.provision "shell", inline: <<-SHELL
                apt-get update && apt-get upgrade -y
                apt-get install golang -y
        SHELL
  end

  config.vm.define "kratos" do |kratos|
        kratos.vm.network "private_network", ip: "192.168.33.91"
        kratos.vm.network "forwarded_port", guest: "7088", host: "7088"

        kratos.vm.provision "shell", inline: <<-SHELL
                apt-get update && apt-get upgrade -y
                apt-get install golang -y
        SHELL
  end
end
```

This configuration creates two servers named `athena` with IP address `192.168.33.90` and `kratos` with IP address `192.168.33.91`. Each server exposes some ports where the services will be listening and is provisioned by installing Go. 

### Athena (Server)
The program define a simple in-memory key-value store (Lines 36-39), with `Get (Lines 41-54)`, `Put (Lines 57-65)` methods (to the exported). Then, there is a `main method (Lines 68-90)` that creates the key-value services, registers it with the RPC, starts a server and listens for requests.
```go
  1 package main
  2
  3 import (
  4     "log"
  5     "net"
  6     "net/rpc"
  7     "sync"
  8 )
  9
 10 const (
 11     OK       = "OK"
 12     ErrNoKey = "ErrNoKey"
 13 )
 14
 15 type Err string
 16
 17 type PutArgs struct {
 18     Key   string
 19     Value string
 20 }
 21
 22 type PutReply struct {
 23     Err Err
 24 }
 25
 26 type GetArgs struct {
 27     Key string
 28 }
 29
 30 type GetReply struct {
 31     Err   Err
 32     Value string
 33 }
 34
 35
 36 type KV struct {
 37     mu sync.Mutex
 38     data map[string]string
 39 }
 40
 41 func (kv *KV) Get(args *GetArgs, reply *GetReply) error {
 42     kv.mu.Lock()
 43     defer kv.mu.Unlock()
 44
 45     val, ok := kv.data[args.Key]; if ok {
 46         reply.Err = OK
 47         reply.Value = val
 48     } else {
 49         reply.Err = ErrNoKey
 50         reply.Value = ""
 51     }
 52
 53     return nil
 54 }
 55
 56
 57 func (kv *KV) Put(args *PutArgs, reply *PutReply) error {
 58     kv.mu.Lock()
 59     defer kv.mu.Unlock()
 60
 61     kv.data[args.Key] = args.Value
 62     reply.Err = OK
 63
 64     return nil
 65 }
 66
 67
 68 func main() {
 69     kv := new(KV)
 70     kv.data = map[string]string{}
 71
 72     rpcs := rpc.NewServer()
 73     rpcs.Register(kv)
 74
 75     l, e := net.Listen("tcp", ":7077")
 76     if e != nil {
 77         log.Fatal("listen error:", e)
 78     }
 79
 80     for {
 81         conn, err := l.Accept()
 82         if err == nil {
 83             go rpcs.ServeConn(conn)
 84         } else {
 85             break
 86         }
 87     }
 88
 89     l.Close()
 90 }
```

### Kratos (Client)
The client defines two eponymous methods, `get (Lines 44-58)` and `put (Lines 60-72)`, in addition to a `connect (Lines 34-41)` that establishes a connection to the server and returns a client handle.
In addition, there are mirror definitions for the argument and return value types of the methods defined in the server. This ensures that the correct arguments are sent and the correct response received.
```go
  1 package main
  2
  3 import (
  4     "fmt"
  5     "log"
  6     "net/rpc"
  7 )
  8
  9 const (
 10     OK       = "OK"
 11     ErrNoKey = "ErrNoKey"
 12 )
 13
 14 type Err string
 15
 16 type PutArgs struct {
 17     Key   string
 18     Value string
 19 }
 20
 21 type PutReply struct {
 22     Err Err
 23 }
 24
 25 type GetArgs struct {
 26     Key string
 27 }
 28
 29 type GetReply struct {
 30     Err   Err
 31     Value string
 32 }
 33
 34 func connect() *rpc.Client {
 35     client, err := rpc.Dial("tcp", "192.168.33.90:7077")
 36     if err != nil {
 37         log.Fatal("dialing:", err)
 38     }
 39
 40     return client
 41 }
 42
 43
 44 func get(key string) string {
 45     client := connect()
 46
 47     args := GetArgs{key}
 48     reply := GetReply{}
 49
 50     err := client.Call("KV.Get", &args, &reply)
 51     if err != nil {
 52         log.Fatal("error:", err)
 53     }
 54
 55     client.Close()
 56
 57     return reply.Value
 58 }
 59
 60 func put(key, value string) {
 61     client := connect()
 62
 63     args := PutArgs{key, value}
 64     reply := PutReply{}
 65
 66     err := client.Call("KV.Put", &args, &reply)
 67     if err != nil {
 68          log.Fatal("error:", err)
 69     }
 70
 71     client.Close()
 72 }
 73
 74
 75 func main() {
 76     put("subject", "6.824")
 77     fmt.Printf("Put(subject, 6.824) done\n")
 78     fmt.Printf("get(subject) -> %s\n", get("subject"))
 79
 80     fmt.Println()
 81
 82     put("foo", "bar")
 83     fmt.Printf("Put(foo, bar) done\n")
 84     fmt.Printf("get(foo) -> %s\n", get("foo"))
 85
 86     fmt.Println()
 87
 88     put("bonnie", "clyde")
 89     fmt.Printf("Put(bonnie, clyde) done\n")
 90     fmt.Printf("get(bonnie) -> %s\n", get("bonnie"))
 91 }
```
The `main function (Lines 75-91)` simply calls the `put` method to store some data in the key-value store on the server and calls `get` to retrieve it.

## Conclusion
This is as simple as RPC gets. For a more production-ready system, more robust frameworks such as gRPC and Apache Thrift should be preferred. gRPC provides a flexible, full-featured interface-definition language called protocol buffers that makes defining services a breeze. It also offers security features and improved performance by supporting bi-directional streaming among others.