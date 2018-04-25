# Microservices in Golang and Micro - Part 1

# Introduction
This is a ten part series, released weekly on writing microservices in Golang. Making use of protobuf and gRPC as the underlying transport protocol. Why? Because it took me a long time to figure this out and settle on a solution that was clear and concise, and I wanted to share what I'd learnt about creating, testing and deploying microservices end-to-end with others new to the scene.

In this tutorial, we will just be covering some of the basic concepts, terminology, and creating our first microservice in its crudest form.

We will be creating the following services throughout the series:

- consignments
- inventory
- users
- authentication
- roles
- vessels

The stack we will end up with will be: golang, mongodb, grpc, docker, Google Cloud, Kubernetes, NATS, CircleCI, Terraform and go-micro.

To follow along, you can follow the steps I include, but be sure to change the GOPATH to your own, using the [git repo](https://github.com/EwanValentine/shippy) (each article has its own branch) as reference.

Also note, I'm working on a Macbook, so you might need to alter your Makefiles to use `$GOPATH` instead of `$(GOPATH)`, and there may be some other inconsistencies between operating systems etc.

## Prerequisites
- A fair understanding of Golang and its ecosystem.
- Install gRPC / protobuf - [see here](https://grpc.io/docs/quickstart/go.html)
- Install Golang - [see here](https://golang.org/doc/install)
- Install the following go libraries:
```
go get -u google.golang.org/grpc
go get -u github.com/golang/protobuf/protoc-gen-go
```

## What are we building?
We will be building perhaps the most generic microservice example you can think of, a shipping container management platform! A blog felt too simple a use-case for microservices, I wanted something that could really show-off the separation of complexity. So this felt like a good challenge!

So let's start with the basics:

## What is a Microservice?
In a traditional monolith application, all of an organisations features are written into one single application. Sometime's they're grouped by their type, such as controllers, entities, factories etc. Other times, perhaps in larger application, features are separated by concern or by feature. So you may have an auth package, a friends package, and an articles package. Which may contain their own set of factories, services, repositories, models etc. But ultimately they are grouped together within a single codebase.

A microservice is the concept of taking that second approach slightly further, and separating those concerns into their own, independent runnable codebase.


## Why microservicess?

Complexity - Splitting features into microservices allows you to split code into smaller chunks. It harks back to the old unix adage of 'doing one thing well'. There's a tendency with monoliths to allow domains to become tightly coupled with one another, and concerns to become blurred. This leads to riskier, more complex updates, potentially more bugs and more difficult integrations.

Scale - In a monolith, certain areas of code may be used more frequently than others. With a monolith, you can only scale the entire codebase. So if your auth service is hit constantly, you need to scale the entire codebase to cope with the load for just your auth service.

With microservices, that separation allows you to scale individual services individually. Meaning more efficient horizontal scaling. Which works very nicely with cloud computing with multiple cores and regions etc.

__Nginx wrote a fantastic series on the various concepts of microservices, [please give this a read](https://www.nginx.com/blog/introduction-to-microservices/).__


## Why Golang?

Microservices are supported by just about all languages, after all, microservices are a concept rather than a specific framework or tool. That being said, some languages are better suited and, or have better support for microservices than others. One language with great support is Golang.

Golang is very light-weight, very fast, and has a fantastic support for concurrency, which is a powerful capability when running across several machines and cores.

Go also contains a very powerful standard library for writing web services.

Finally, there is a fantastic microservice framework available for Go called go-micro. Which we will be using in this series.


## Introducing protobuf/gRPC
Because microservices are split out into separate codebases, one important issue with microservices, is communication. In a monolith communication is not an issue, as you call code directly from elsewhere in your codebase. However, microservices don't have that ability, as they live in separate places. So you need a way in which these independent services can talk to one another with as little latency as possible.

Here, you could use traditional REST, such as JSON or XML over http. However, the problem with this approach is that service A has to encode its data into JSON/XML, send a large string over the wire, to service B, which then has to decode this message from JSON, back into code. This has potential overhead problems at scale. Whilst you're forced to adopt this form of communication for web browsers, services can just about talk to each other in any format they wish.

In comes [gRPC](https://grpc.io/). [gRPC](https://grpc.io/) is a light-weight binary based RPC communication protocol brought out by Google. That's a lot of words, so let's dissect that a little. gRPC uses binary as its core data format. In our RESTful example, using JSON, you would be sending a string over http. Strings contain bulky metadata about its encoding format; about its length, its content format and various other bits and pieces. This is so that a server can inform a traditionally browser based client what to expect. We don't really need all of this when communicating between two services. So we can use cold hard binary, which is much more light-weight. gRPC uses the new HTTP 2.0 spec, which allows for the use of binary data. It even allows for bi-directional streaming, which is pretty cool! HTTP 2 is pretty fundamental to how gRPC works. For more on HTTP 2, [take a look at this fantastic post from Google](https://developers.google.com/web/fundamentals/performance/http2/).

But how can we do anything with binary data? Well, gRPC has an interchange DSL called protobuf. Protobuf allows you to define an interface to your service using a developer friendly format.

So let's start by creating our first service definition. Create the following file: `consignment-service/proto/consignment/consignment.proto` from the root directory of our repo. For the time being, I'm housing all of our services in a single repo. This is known as a mono-repo. This is mostly to keep things simple for this tutorial. There are many arguments for and against using mono-repos, which I won't go into here. You could house all of these services and components in separate repos, there are many good arguments for that approach also.

[Here's a fantastic article on gRPC](https://blog.gopheracademy.com/advent-2017/go-grpc-beyond-basics/) I highly recommend you give it a read.

In the consignment.proto file you just created, add the following:

```
// consignment-service/proto/consignment/consignment.proto
syntax = "proto3";

package go.micro.srv.consignment;

service ShippingService {
  rpc CreateConsignment(Consignment) returns (Response) {}
}

message Consignment {
  string id = 1;
  string description = 2;
  int32 weight = 3;
  repeated Container containers = 4;
  string vessel_id = 5;
}

message Container {
  string id = 1;
  string customer_id = 2;
  string origin = 3;
  string user_id = 4;
}

message Response {
  bool created = 1;
  Consignment consignment = 2;
}
```

This is a really basic example, but there are a few things going on here. First of all, you define your service, this should contain the methods that you wish to expose to other services. Then you define your message types, these are effectively your data structure. Protobuf is statically typed, and you can define custom types, as we have done with `Container`. Messages are themselves just custom types.

There are two libraries at work here, messages are handled by protobuf, and the service we defined is handled by a gRPC protobuf plugin, which compiles code to interact with these types, i.e the `service` part of our proto file.

This protobuf definition is then ran through a CLI to generate the code to interface this binary data and your functionality.

Speaking of which, let's create a Makefile for our first service `$ touch consignment-service/Makefile`.

*Note: be careful with the formatting when copying the Makefile code, they have to be tab spaced, otherwise it will break. Make sure your editor has linting or is set-up properly for Makefiles.*

```
build:
	protoc -I. --go_out=plugins=grpc:$(GOPATH)/src/github.com/ewanvalentine/shipper/consignment-service \
	  proto/consignment/consignment.proto
```
This will call the protoc library, which is responsible for compiling your protobuf definition into code. We also specify the use of the grpc plugin, as well as the build context and the output path.

Now, when you run `$ make build` from this services directory, and look in `proto/consignment/` you should see a new Go file, called `consignment.pb.go`. This is code automatically generated by the gRPC/protobuf libraries to allow you to interface your protobuf definition to your own code.


So let's set that up now. Create your main.go file `$ touch consignment-service/main.go` from the project root.

```go
// consignment-service/main.go
package main

import (
	"log"
	"net"

        // Import the generated protobuf code
	pb "github.com/ewanvalentine/shipper/consignment-service/proto/consignment"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

const (
	port = ":50051"
)

type IRepository interface {
	Create(*pb.Consignment) (*pb.Consignment, error)
}

// Repository - Dummy repository, this simulates the use of a datastore
// of some kind. We'll replace this with a real implementation later on.
type Repository struct {
	consignments []*pb.Consignment
}

func (repo *Repository) Create(consignment *pb.Consignment) (*pb.Consignment, error) {
	updated := append(repo.consignments, consignment)
	repo.consignments = updated
	return consignment, nil
}

// Service should implement all of the methods to satisfy the service
// we defined in our protobuf definition. You can check the interface
// in the generated code itself for the exact method signatures etc
// to give you a better idea.
type service struct {
	repo IRepository
}

// CreateConsignment - we created just one method on our service,
// which is a create method, which takes a context and a request as an
// argument, these are handled by the gRPC server.
func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment) (*pb.Response, error) {

        // Save our consignment
	consignment, err := s.repo.Create(req)
	if err != nil {
		return nil, err
	}

        // Return matching the `Response` message we created in our
        // protobuf definition.
	return &pb.Response{Created: true, Consignment: consignment}, nil
}

func main() {

	repo := &Repository{}

        // Set-up our gRPC server.
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()

        // Register our service with the gRPC server, this will tie our
        // implementation into the auto-generated interface code for our
        // protobuf definition.
	pb.RegisterShippingServiceServer(s, &service{repo})

	// Register reflection service on gRPC server.
	reflection.Register(s)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

Please read the comments left in the code carefully. But in summary, here we are creating the implementation logic which our gRPC methods interface with, using the generated formats, creating a new gRPC server on port 50051. There you have it! A fully functional gRPC service. You can run this with `$ go run main.go`, but you won't see anything, and you won't be able to use it yet... So let's create a client to see it in action.

Let's create a command line interface, which will take a JSON consignment file and interact with our gRPC service.

In your root directory, create a new sub-directory `$ mkdir consignment-cli`. In that directory, create a file called `cli.go`, with the following content:

```go
// consignment-cli/cli.go
package main

import (
	"encoding/json"
	"io/ioutil"
	"log"
	"os"

	pb "github.com/ewanvalentine/shipper/consignment-service/proto/consignment"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
)

const (
	address         = "localhost:50051"
	defaultFilename = "consignment.json"
)

func parseFile(file string) (*pb.Consignment, error) {
	var consignment *pb.Consignment
	data, err := ioutil.ReadFile(file)
	if err != nil {
		return nil, err
	}
	json.Unmarshal(data, &consignment)
	return consignment, err
}

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Did not connect: %v", err)
	}
	defer conn.Close()
	client := pb.NewShippingService(conn)

	// Contact the server and print out its response.
	file := defaultFilename
	if len(os.Args) > 1 {
		file = os.Args[1]
	}

	consignment, err := parseFile(file)

	if err != nil {
		log.Fatalf("Could not parse file: %v", err)
	}

	r, err := client.CreateConsignment(context.Background(), consignment)
	if err != nil {
		log.Fatalf("Could not greet: %v", err)
	}
	log.Printf("Created: %t", r.Created)
}

```

Now create a consignment (consignment-cli/consignment.json):

```js
{
  "description": "This is a test consignment",
  "weight": 550,
  "containers": [
    { "customer_id": "cust001", "user_id": "user001", "origin": "Manchester, United Kingdom" }
  ],
  "vessel_id": "vessel001"
}
```


Now if you run `$ go run main.go` in `consignment-service`, and then in a separate terminal pane, run `$ go run cli.go`. You should see a message saying `Created: true`. But how can we really check it has created something? Let's update our service with a `GetConsignments` method, so that we can view all of our created consignments.

First let's update our proto definition (I've left comments to denote the changes made):

```protobuf
// consignment-service/proto/consignment/consignment.proto
syntax = "proto3";

package go.micro.srv.consignment;

service ShippingService {
  rpc CreateConsignment(Consignment) returns (Response) {}

  // Created a new method
  rpc GetConsignments(GetRequest) returns (Response) {}
}

message Consignment {
  string id = 1;
  string description = 2;
  int32 weight = 3;
  repeated Container containers = 4;
  string vessel_id = 5;
}

message Container {
  string id = 1;
  string customer_id = 2;
  string origin = 3;
  string user_id = 4;
}

// Created a blank get request
message GetRequest {}

message Response {
  bool created = 1;
  Consignment consignment = 2;

  // Added a pluralised consignment to our generic response message
  repeated Consignment consignments = 3;
}
```

So here we've created a new method on our service called `GetConsignments`, we have also created a new `GetRequest` which doesn't contain anything for the time being. We've also added a `consignments` field to our response message. You will notice the type here has the keyword `repeated` before the actual type. This, as you'd probably have guessed, just means treat this field as an array of these types.

Now run `$ make build` again. Now, try running your service again, you should see an error similar to: `*service does not implement go_micro_srv_consignment.ShippingServiceServer (missing GetConsignments method)`.

Because the implementation of our gRPC methods, are based on matching the interface generated by the protobuf library, we need to ensure our implementation matches our proto definition.

So let's update our `consignment-service/main.go` file:

```go
package main

import (
	"log"
	"net"

	// Import the generated protobuf code
	pb "github.com/ewanvalentine/shipper/consignment-service/proto/consignment"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

const (
	port = ":50051"
)

type IRepository interface {
	Create(*pb.Consignment) (*pb.Consignment, error)
	GetAll() []*pb.Consignment
}

// Repository - Dummy repository, this simulates the use of a datastore
// of some kind. We'll replace this with a real implementation later on.
type Repository struct {
	consignments []*pb.Consignment
}

func (repo *Repository) Create(consignment *pb.Consignment) (*pb.Consignment, error) {
	updated := append(repo.consignments, consignment)
	repo.consignments = updated
	return consignment, nil
}

func (repo *Repository) GetAll() []*pb.Consignment {
	return repo.consignments
}

// Service should implement all of the methods to satisfy the service
// we defined in our protobuf definition. You can check the interface
// in the generated code itself for the exact method signatures etc
// to give you a better idea.
type service struct {
	repo IRepository
}

// CreateConsignment - we created just one method on our service,
// which is a create method, which takes a context and a request as an
// argument, these are handled by the gRPC server.
func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment) (*pb.Response, error) {

	// Save our consignment
	consignment, err := s.repo.Create(req)
	if err != nil {
		return nil, err
	}

	// Return matching the `Response` message we created in our
	// protobuf definition.
	return &pb.Response{Created: true, Consignment: consignment}, nil
}

func (s *service) GetConsignments(ctx context.Context, req *pb.GetRequest) (*pb.Response, error) {
	consignments := s.repo.GetAll()
	return &pb.Response{Consignments: consignments}, nil
}

func main() {

	repo := &Repository{}

	// Set-up our gRPC server.
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()

	// Register our service with the gRPC server, this will tie our
	// implementation into the auto-generated interface code for our
	// protobuf definition.
	pb.RegisterShippingServiceServer(s, &service{repo})

	// Register reflection service on gRPC server.
	reflection.Register(s)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

Here, we have included our new `GetConsignments` method, updated our repository and interface and that satisfies the interface generated by the proto definition. If you run `$ go run main.go` again, this should work again.

Let's update our cli tool to include the ability to call this method and list our consignments:

```go
func main() {
    ...

	getAll, err := client.GetConsignments(context.Background(), &pb.GetRequest{})
	if err != nil {
		log.Fatalf("Could not list consignments: %v", err)
	}
	for _, v := range getAll.Consignments {
		log.Println(v)
	}
}
```


At the very bottom of our main function, underneath where we log out our "Created: success" message, append the code above, and re-run `$ go run cli.go`. This will create a consignment, then call `GetConsignments` after. you should see this list grow the more times you run it.

*Note: for brevity, I may sometimes redact code previously written with a ... to denote no changes were made to the previous code, but additional lines were added or appended.*

So there you have it, we have successfully created a microservice and a client to interact with it, using protobuf and gRPC.

The next part in this series will be around integrating [go-micro](https://github.com/micro/go-micro), which is a powerful framework for creating gRPC based microservices. We will also create our second service, our container service. Speaking of containers, just to confuse matters, we will also look at running our services in Docker containers in the next part in this series.



## Further reading

__Article and newsletters__
https://www.nginx.com/blog/introduction-to-microservices/
https://martinfowler.com/articles/microservices.html
https://www.microservices.com/talks/
https://medium.facilelogin.com/ten-talks-on-microservices-you-cannot-miss-at-any-cost-7bbe5ab7f43f#.ui0748oat
https://microserviceweekly.com/

__Books__
https://www.amazon.co.uk/Building-Microservices-Sam-Newman/dp/1491950358
https://www.amazon.co.uk/Devops-Handbook-World-Class-Reliability-Organizations/dp/1942788002
https://www.amazon.co.uk/Phoenix-Project-DevOps-Helping-Business/dp/0988262509

__Podcasts__
https://softwareengineeringdaily.com/tag/microservices/
https://martinfowler.com/tags/podcast.html
https://www.infoq.com/microservices/podcasts/
