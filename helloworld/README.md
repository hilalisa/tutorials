# Hello World

This guide teaches you how to write a [go-micro](https://github.com/micro/go-micro) service.

By the end you should have a working service and client.

## Dependencies

The following dependencies are required

### Protobuf

Install protoc-gen-micro and it's dependencies for protobuf code generation

- [protoc-gen-micro](https://github.com/micro/protoc-gen-micro)

### Service Discovery

Go-micro uses service discovery to resolve names to addresses. [Consul](https://www.consul.io/) is the default.

Install [consul](https://www.consul.io/downloads.html) and run with:

```
consul agent -dev
```

Alternatively use multicast DNS for zero dependencies. Just specify `MICRO_REGISTRY=mdns` to all your commands

```
MICRO_REGISTRY=mdns go run service.go
```

## Service

The [Service](https://godoc.org/github.com/micro/go-micro#Service) interface is our building block. 
It wraps all the underlying packages of go-micro into an easy to use service definition.

```go
type Service interface {
    Init(...Option)
    Options() Options
    Client() client.Client
    Server() server.Server
    Run() error
    String() string
}
```

### 1. Create a Service

Create a service using `micro.NewService`

```go
import (
	"github.com/micro/go-micro"
)

func main() {
	service := micro.NewService() 
}

```

Options can be passed in during creation set name, version 

```go
service := micro.NewService(
        micro.Name("greeter"),
        micro.Version("latest"),
)
```

All the available options can be found [here](https://godoc.org/github.com/micro/go-micro#Option).

Go Micro also provides a way to set command line flags using `micro.Flags`.

```go
import (
        "github.com/micro/cli"
        "github.com/micro/go-micro"
)

service := micro.NewService(
        micro.Flags(
                cli.StringFlag{
                        Name:  "environment",
                        Usage: "The environment",
                },
        )
)
```

To parse flags use `service.Init`. Additionally access flags use the `micro.Action` option.

```go
service.Init(
        micro.Action(func(c *cli.Context) {
                env := c.StringFlag("environment")
                if len(env) > 0 {
                        fmt.Println("Environment set to", env)
                }
        }),
)
```

Go Micro provides predefined flags which are set and parsed if `service.Init` is called. See all the flags 
[here](https://godoc.org/github.com/micro/go-micro/cmd#pkg-variables).

### 2. Define the API

We use protobuf files to define the service API interface. This is a very convenient way to strictly define the API and 
provide concrete types for both the server and a client.

Here's an example definition.

greeter.proto

```
syntax = "proto3";

service Greeter {
	rpc Hello(HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
	string name = 1;
}

message HelloResponse {
	string greeting = 2;
}
```

Here we're defining a service handler called Greeter with the method Hello which takes the parameter HelloRequest type and returns HelloResponse.

### 3. Generate the API interface

```
protoc --proto_path=$GOPATH/src:. --micro_out=. --go_out=. greeter.proto
```

The types generated can now be imported and used within a **handler** for a server or the client when making a request.

Here's part of the generated code.

```go
type HelloRequest struct {
	Name string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
}

type HelloResponse struct {
	Greeting string `protobuf:"bytes,2,opt,name=greeting" json:"greeting,omitempty"`
}

// Client API for Greeter service

type GreeterClient interface {
	Hello(ctx context.Context, in *HelloRequest, opts ...client.CallOption) (*HelloResponse, error)
}

type greeterClient struct {
	c           client.Client
	serviceName string
}

func NewGreeterClient(serviceName string, c client.Client) GreeterClient {
	if c == nil {
		c = client.NewClient()
	}
	if len(serviceName) == 0 {
		serviceName = "greeter"
	}
	return &greeterClient{
		c:           c,
		serviceName: serviceName,
	}
}

func (c *greeterClient) Hello(ctx context.Context, in *HelloRequest, opts ...client.CallOption) (*HelloResponse, error) {
	req := c.c.NewRequest(c.serviceName, "Greeter.Hello", in)
	out := new(HelloResponse)
	err := c.c.Call(ctx, req, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

// Server API for Greeter service

type GreeterHandler interface {
	Hello(context.Context, *HelloRequest, *HelloResponse) error
}

func RegisterGreeterHandler(s server.Server, hdlr GreeterHandler) {
	s.Handle(s.NewHandler(&Greeter{hdlr}))
}
```

### 4. Implement the handler

The server requires **handlers** to be registered to serve requests. A handler is an public type with public methods 
which conform to the signature `func(ctx context.Context, req interface{}, rsp interface{}) error`.

As you can see above, a handler signature for the Greeter interface looks like so.

```go
type GreeterHandler interface {
        Hello(context.Context, *HelloRequest, *HelloResponse) error
}
```

Here's an implementation of the Greeter handler.

```go
import proto "github.com/micro/examples/helloworld/proto"

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
	rsp.Greeting = "Hello " + req.Name
	return nil
}
```


The handler is registered with your service much like a http.Handler.

```
service := micro.NewService(
	micro.Name("greeter"),
)

proto.RegisterGreeterHandler(service.Server(), new(Greeter))
```

You can also create a bidirectional streaming handler but we'll leave that for another day.

### 5. Running the service

The service can be run by calling `server.Run`. This causes the service to bind to the address in the config 
(which defaults to the first RFC1918 interface found and a random port) and listen for requests.

This will additionally Register the service with the registry on start and Deregister when issued a kill signal.

```go
if err := service.Run(); err != nil {
	log.Fatal(err)
}
```

### 6. The complete service

<br>
greeter.go

```go
package main

import (
        "context"
        "log"

        "github.com/micro/go-micro"
        proto "github.com/micro/examples/helloworld/proto"
)

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
        rsp.Greeting = "Hello " + req.Name
        return nil
}

func main() {
        service := micro.NewService(
                micro.Name("greeter"),
        )

        service.Init()

        proto.RegisterGreeterHandler(service.Server(), new(Greeter))

        if err := service.Run(); err != nil {
                log.Fatal(err)
        }
}
```

## Call Service

### CLI

Use the micro CLI to query the service

```
go get github.com/micro/micro
```

```
micro query greeter Greeter.Hello '{"name": "john"}'
```

### Client

The [client](https://godoc.org/github.com/micro/go-micro/client) package is used to query services. When you create a 
Service, a Client is included which matches the initialised packages used by the server.

Querying the above service is as simple as the following.

```go
service := micro.NewService()
service.Init()

greeter := proto.NewGreeterClient("greeter", service.Client())

rsp, err := greeter.Hello(context.TODO(), &proto.HelloRequest{
	Name: "John",
})
if err != nil {
	fmt.Println(err)
	return
}

fmt.Println(rsp.Greeter)
```

## Code

The full example can be found at [examples/helloworld](https://github.com/micro/examples/tree/master/helloworld).

