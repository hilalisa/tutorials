# Microservices with Golang and Micro - Part 2

# Introduction - Part 2 Docker and go-micro
[In the previous post](https://ewanvalentine.io/microservices-in-golang-part-1/), we covered the basics of writing a gRPC based microservice. In this part; we will cover the basics of Dockerising a service, we will also be updating our service to use [go-micro](https://github.com/micro/go-micro), and finally, introducing a second service.

# Introducing Docker.
With the advent of cloud computing, and the birth of microservices. The pressure to deploy more, but smaller chunks of code at a time has led to some interesting new ideas and technologies. One of which being the concept of [containers](https://en.wikipedia.org/wiki/Operating-system-level_virtualization).


Traditionally, teams would deploy a monolith to static servers, running a set operating system, with a predefined set of dependencies to keep track of. Or maybe on a VM provisioned by Chef or Puppet for example. Scaling was expensive and not all that effective. The most common option was vertical scaling, i.e throwing more and more resources at static servers.

Tools like [vagrant](https://www.vagrantup.com/) came along and made provisioning VM's fairly trivial. But running a VM was still a fairly heft operation. You were running a full operating system in all its glory, kernel and all, within your host machine. In terms of resources, this is pretty expensive. So when microservices came along, it became infeasible to run so many separate codebases in their own environments.

## Along came containers
[Containers](https://en.wikipedia.org/wiki/Operating-system-level_virtualization) are slimmed down versions of an operating system. Containers don't contain a kernel, a guest OS or any of the lower level components which would typically make up an OS.

Containers only contain the top level libraries and its run-time components. The kernel is shared across the host machine. So the host machine runs a single Unix kernel, which is then shared by n amount of containers, running very different sets of run-times.

Under the hood, containers utilise various kernel utilities, in order to share resources and network functionality across the container space.

[Further reading](https://www.redhat.com/en/topics/containers/whats-a-linux-container)


This means you can run the run-time and the dependencies your code needs, without booting up several complete operating systems. This is a game changer because the overall size of a container vs a VM, is magnitudes smaller. Ubuntu for example, is typically a little under 1GB in size. Whereas its Docker image counterpart is a mere 188mb.

You will notice I spoke more broadly of containers in that introduction, rather than 'Docker containers'. It's common to think that [Docker](https://www.docker.com/) and containers are the same thing. However, containers are more of a concept or set of capabilities within Linux. [Docker](https://www.docker.com/) is just a flavor of containers, which became popular largely due to its ease of use. There are [others](https://www.contino.io/insights/beyond-docker-other-types-of-containers), too. But we'll be sticking with Docker as it's in my opinion the best supported, and the simplest for newcomers.

So now hopefully you see the value in containerisation, we can start Dockerising our first service. Let's create a Dockerfile `$ touch consignment-service/Dockerfile`.

In that file, add the following:

```
FROM alpine:latest

RUN mkdir /app
WORKDIR /app
ADD consignment-service /app/consignment-service

CMD ["./consignment-service"]
```

If you're running on Linux, you might run into issues using Alpine. So if you're following this article on a Linux machine, simply replace `alpine` with `debian`, and you should be good to go. We'll touch on an even better way to build our binaries later on.

First of all, we are pulling in the latest [Linux Alpine](https://alpinelinux.org/) image. [Linux Alpine](https://alpinelinux.org/) is a light-weight Linux distribution, developed and optimised for running Dockerised web applications. In other words, [Linux Alpine](https://alpinelinux.org/) has just enough dependencies and run-time functionality to run most applications. This means its image size is around 8mb(!!). Which compared with say... an Ubuntu VM at around 1GB, you can start to see why Docker images became a more natural fit for microservices and cloud computing.

Next we create a new directory to house our application, and set the context directory to our new directory. This is so that our app directory is the default directory. We then add our compiled binary into our Docker container, and run it.

Now let's update our Makefile's build entry to build our docker image.

```
build:
    ...
    GOOS=linux GOARCH=amd64 go build
    docker build -t consignment-service .
```

We've added two more steps here, and I'd like to explain them in a little more detail. First of all, we're building our Go binary. You will notice two environment variables are being set before we run `$ go build` however. GOOS and GOARCH allow you to cross-compile your go binary for another operating system. Since I'm developing on a Macbook, I can't compile a go binary, and then run it within a Docker container, which uses Linux. The binary will be completely meaningless within your Docker container and it will throw an error.

The second step I added is the docker build process. This will read your Dockerfile, and build an image by the name `consignment-service`, the period denotes a directory path, so here we just want the build process to look in the current directory.

I'm going to add a new entry in our Makefile:

```
run:
    docker run -p 50051:50051 consignment-service
```

Here, we run our consignment-service docker image, exposing the port 50051. Because Docker runs on a separate networking layer, you need to forward the port used within your Docker container, to your host. You can forward the internal port to a new port on the host by changing the first segment. For example, if you wanted to run this service on port 8080, you would change the -p argument to `8080:50051`. You can also run a container in the background by including a `-d` flag. For example `docker run -d -p 50051:50051 consignment-service`.

[You can read more about how Docker's networking works here](https://docs.docker.com/engine/userguide/networking/).


Run `$ make run`, then in a separate terminal pane, run your cli client again `$ go run cli.go` and double check it still works.

When you run `$ docker build`, you are building your code and run-time environment into an image. Docker images are portable snapshots of your environment, its dependencies. You can share docker images by publishing them to docker hub. Which is like a sort of npm, or yum repo for docker images. When you define a `FROM` in your Dockerfile, you are telling docker to pull that image from docker hub to use as your base. You can then extend and override parts of that base file, by re-defining them in your own. We won't be publishing our docker images, but feel free to peruse [docker hub](https://hub.docker.com/explore/), and note how just about any piece of software has been containerised already. Some really [remarkable things](https://www.youtube.com/watch?v=GsLZz8cZCzc) have been Dockerised.

Each declaration within a Dockerfile is cached when it's first built. This saves having to re-build the entire run-time each time you make a change. Docker is clever enough to work out which parts have changed, and which parts needs re-building. This makes the build process incredibly quick.

Enough about containers! Let's get back to our code.

When creating a gRPC service, there's quite a lot of boilerplate code for creating connections, and you have to hard-code the location of the service address into a client, or other service in order for it to connect to it. This is tricky, because when you are running services in the cloud, they may not share the same host, or the address or ip may change after re-deploying a service.

This is where service discovery comes into play. Service discovery keeps an up-to-date catalogue of all your services and their locations. Each service registers itself at runtime, and de-registers itself on closure. Each service then has a name or id assigned to it. So that even though it may have a new IP address, or host address, as long as the service name remains the same, you don't need to update calls to this service from your other services.

Typically, there are many approaches to this problem, but like most things in programming, if someone has tackled this problem already, there's no point re-inventing the wheel. One person who has tackled these problems with fantastic clarity and ease of use, is @chuhnk (Asim Aslam), creator of [Go-micro](https://github.com/micro/go-micro).

## Go-micro
Go-micro is a powerful microservice framework written in Go, for use, for the most part with Go. However you can use [Sidecar](https://github.com/micro/micro/tree/master/car) in order to interface with other languages also.

Go-micro has useful features for making microservices in Go trivial. But we'll start with probably the most common issue it solves, and that's service discovery.

We will need to make a few updates to our service in order to work with go-micro. Go-micro integrates as a protoc plugin, in this case replacing the standard gRPC plugin we're currently using. So let's start by replacing that in our Makefile.

*Be sure to install the go-micro dependencies:*
```
go get -u github.com/micro/protobuf/{proto,protoc-gen-go}
```

```
build:
	protoc -I. --go_out=plugins=micro:$(GOPATH)/src/github.com/EwanValentine/shippy/consignment-service \
		proto/consignment/consignment.proto
	...

...
```

We have updated our Makefile to use the go-micro plug-in, instead of the gRPC plugin. Now we will need to update our `consignment-service/main.go` file to use go-micro. This will abstract much of our previous gRPC code. It handles registering and spinning up our service with ease.

```go
// consignment-service/main.go
package main

import (

	// Import the generated protobuf code
	"fmt"

	pb "github.com/EwanValentine/shippy/consignment-service/proto/consignment"
	micro "github.com/micro/go-micro"
	"golang.org/x/net/context"
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
func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment, res *pb.Response) error {

	// Save our consignment
	consignment, err := s.repo.Create(req)
	if err != nil {
		return err
	}

	// Return matching the `Response` message we created in our
	// protobuf definition.
	res.Created = true
	res.Consignment = consignment
	return nil
}

func (s *service) GetConsignments(ctx context.Context, req *pb.GetRequest, res *pb.Response) error {
	consignments := s.repo.GetAll()
	res.Consignments = consignments
	return nil
}

func main() {

	repo := &Repository{}

	// Create a new service. Optionally include some options here.
	srv := micro.NewService(

		// This name must match the package name given in your protobuf definition
		micro.Name("go.micro.srv.consignment"),
		micro.Version("latest"),
	)

	// Init will parse the command line flags.
	srv.Init()

	// Register handler
	pb.RegisterShippingServiceHandler(srv.Server(), &service{repo})

	// Run the server
	if err := srv.Run(); err != nil {
		fmt.Println(err)
	}
}

```

The main changes here are the way in which we instantiate our gRPC server, which has been abstracted neatly behind a `mico.NewService()` method, which handles registering our service. And finally, the `service.Run()` function, which handles the connection itself. Similar as before, we register our implementation, but this time using a slightly different method.

The second biggest changes are to the service methods themselves, the arguments and response types have changes slightly to take both the request and the response structs as arguments, and now only returning an error. Within our methods, we set the response, which is handled by go-micro.

Finally, we are no longer hard-coding the port. Go-micro should be configured using environment variables, or command line arguments. To set the address, use `MICRO_SERVER_ADDRESS=:50051`. We also need to tell our service to use [mdns](https://en.wikipedia.org/wiki/Multicast_DNS) (multicast dns) as our service broker for local use. You wouldn't typically use [mdns](https://en.wikipedia.org/wiki/Multicast_DNS) for service discovery in production, but we want to avoid having to run something like Consul or etcd locally for the sakes of testing. More on this in a later post.

Let's update our Makefile to reflect this.

```
run:
    docker run -p 50051:50051 \
        -e MICRO_SERVER_ADDRESS=:50051 \
        -e MICRO_REGISTRY=mdns consignment-service
```

The `-e` is an environment variable flag, this allows you to pass in environment variables into your Docker container. You must have a flag per variable, for example `-e ENV=staging -e DB_HOST=localhost` etc.

Now if you run `$ make run`, you will have a Dockerised service, with service discovery. So let's update our cli tool to utilise this.

```go
import (
    ...
    "github.com/micro/go-micro/cmd"
    microclient "github.com/micro/go-micro/client"

)

func main() {
    cmd.Init()

    // Create new greeter client
    client := pb.NewShippingService("go.micro.srv.consignment", microclient.DefaultClient)
    ...
}
```

[See here for full file](https://github.com/EwanValentine/shippy/blob/tutorial-2/consignment-cli/cli.go)

Here we've imported the go-micro libraries for creating clients, and replaced our existing connection code, with the go-micro client code, which uses service resolution instead of connecting directly to an address.

However if you run this, this won't work. This is because we're running our service in a Docker container now, which has its own [mdns](https://en.wikipedia.org/wiki/Multicast_DNS), separate to the host [mdns](https://en.wikipedia.org/wiki/Multicast_DNS) we are currently using. The easiest way to fix this is to ensure both service and client are running in "dockerland", so that they are both running on the same host, and using the same network layer. So let's create a Makefile `consignment-cli/Makefile`, and create some entries.

```
build:
	GOOS=linux GOARCH=amd64 go build
	docker build -t consignment-cli .

run:
	docker run -e MICRO_REGISTRY=mdns consignment-cli
```
Similar to before, we want to build our binary for Linux. When we run our docker image, we want to pass in an environment variable to instruct go-micro to use mdns.

Now let's create a Dockerfile for our CLI tool:

```
FROM alpine:latest

RUN mkdir -p /app
WORKDIR /app

ADD consignment.json /app/consignment.json
ADD consignment-cli /app/consignment-cli

CMD ["./consignment-cli"]
```

This is very similar to our services Dockerfile, except it also pulls in our json data file as well.

Now when you run `$ make run` in your `consignment-cli` directory, you should see `Created: true`, the same as before.

Earlier, I mentioned that those of you using Linux should switch to use the Debian base image. Now seems like a good time to take a look at a new feature from Docker: Multi-stage builds. This allows us to use multiple Docker images in one Dockerfile.

This is useful in our case especially, as we can use one image to build our binary, with all the correct dependencies etc, then use the second image to run it. Let's try this out, I will leave detailed comments along-side the code:

```
# consignment-service/Dockerfile

# We use the official golang image, which contains all the
# correct build tools and libraries. Notice `as builder`,
# this gives this container a name that we can reference later on.
FROM golang:1.9.0 as builder

# Set our workdir to our current service in the gopath
WORKDIR /go/src/github.com/EwanValentine/shippy/consignment-service

# Copy the current code into our workdir
COPY . .

# Here we're pulling in godep, which is a dependency manager tool,
# we're going to use dep instead of go get, to get around a few
# quirks in how go get works with sub-packages.
RUN go get -u github.com/golang/dep/cmd/dep

# Create a dep project, and run `ensure`, which will pull in all
# of the dependencies within this directory.
RUN dep init && dep ensure

# Build the binary, with a few flags which will allow
# us to run this binary in Alpine.
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo .

# Here we're using a second FROM statement, which is strange,
# but this tells Docker to start a new build process with this
# image.
FROM alpine:latest

# Security related package, good to have.
RUN apk --no-cache add ca-certificates

# Same as before, create a directory for our app.
RUN mkdir /app
WORKDIR /app

# Here, instead of copying the binary from our host machine,
# we pull the binary from the container named `builder`, within
# this build context. This reaches into our previous image, finds
# the binary we built, and pulls it into this container. Amazing!
COPY --from=builder /go/src/github.com/EwanValentine/shippy/consignment-service/consignment-service .

# Run the binary as per usual! This time with a binary build in a
# separate container, with all of the correct dependencies and
# run time libraries.
CMD ["./consignment-service"]
```

The only issue with this approach, and I would like to come back and improve this at some point, is that Docker cannot read files from a parent directory. It can only read files from the same directory, or subdirectories of where the Dockerfile lives.

This means that in order to run `$ dep ensure` or `$ go get`, you will need to ensure you have your code pushed up to Git, so that it can pull in the vessel-service for example. Just as you would any other Go package. Not ideal, but good enough for now.

I will now go through our other Dockerfiles and apply this new approach. Oh, and remember to remember to remove `$ go build` from your Makefiles!

[More on multi-stage builds here.](https://docs.docker.com/engine/userguide/eng-image/multistage-build/#name-your-build-stages)

# Vessel service
Let's create a second service. We have a consignment service, this will deal with matching a consignment of containers to a vessel which is best suited to that consignment. In order to match our consignment, we need to send the weight and amount of containers to our new vessel service, which will then find a vessel capable of handling that consignment.

Create a new directory in your root directory `$ mkdir vessel-service`, now created a sub-directory for our new services protobuf definition, `$ mkdir -p vessel-service/proto/vessel`. Now let's create a new protobuf file, `$ touch vessel-service/proto/vessel/vessel.proto`.

Since the protobuf definition is really the core of our domain design, let's start there.

```
// vessel-service/proto/vessel/vessel.proto
syntax = "proto3";

package go.micro.srv.vessel;

service VesselService {
  rpc FindAvailable(Specification) returns (Response) {}
}

message Vessel {
  string id = 1;
  int32 capacity = 2;
  int32 max_weight = 3;
  string name = 4;
  bool available = 5;
  string owner_id = 6;
}

message Specification {
  int32 capacity = 1;
  int32 max_weight = 2;
}

message Response {
  Vessel vessel = 1;
  repeated Vessel vessels = 2;
}
```

As you can see, this is very similar to our first service. We create a service, with a single rpc method called `FindAvailable`. This takes a `Specification` type and returns a `Response` type. The `Response` type returns either a `Vessel` type or multiple Vessels, using the repeated field.

Now we need to create a Makefile to handle our build logic and our run script. `$ touch vessel-service/Makefile`. Open that file and add the following:

```
// vessel-service/Makefile
build:
	protoc -I. --go_out=plugins=micro:$(GOPATH)/src/github.com/EwanValentine/shippy/vessel-service \
    proto/vessel/vessel.proto
	docker build -t vessel-service .

run:
	docker run -p 50052:50051 -e MICRO_SERVER_ADDRESS=:50051 -e MICRO_REGISTRY=mdns vessel-service

```

This is almost identical to the first Makefile we created for our consignment-service, however notice the service names and the ports have changed a little. We can't run two docker containers on the same port, so we make use of Dockers port forwarding here to ensure this service forwards 50051 to 50052 on the host network.

Now we need a Dockerfile, using our new multi-stage format:

```
# vessel-service/Dockerfile
FROM golang:1.9.0 as builder

WORKDIR /go/src/github.com/EwanValentine/shippy/vessel-service

COPY . .

RUN go get -u github.com/golang/dep/cmd/dep
RUN dep init && dep ensure
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo .


FROM alpine:latest

RUN apk --no-cache add ca-certificates

RUN mkdir /app
WORKDIR /app
COPY --from=builder /go/src/github.com/EwanValentine/shippy/vessel-service/vessel-service .

CMD ["./vessel-service"]

```

Finally, we can start on our implementation:

```go
// vessel-service/main.go
package main

import (
	"context"
	"errors"
	"fmt"

	pb "github.com/EwanValentine/shippy/vessel-service/proto/vessel"
	"github.com/micro/go-micro"
)

type Repository interface {
	FindAvailable(*pb.Specification) (*pb.Vessel, error)
}

type VesselRepository struct {
	vessels []*pb.Vessel
}

// FindAvailable - checks a specification against a map of vessels,
// if capacity and max weight are below a vessels capacity and max weight,
// then return that vessel.
func (repo *VesselRepository) FindAvailable(spec *pb.Specification) (*pb.Vessel, error) {
	for _, vessel := range repo.vessels {
		if spec.Capacity <= vessel.Capacity && spec.MaxWeight <= vessel.MaxWeight {
			return vessel, nil
		}
	}
	return nil, errors.New("No vessel found by that spec")
}

// Our grpc service handler
type service struct {
	repo Repository
}

func (s *service) FindAvailable(ctx context.Context, req *pb.Specification, res *pb.Response) error {

	// Find the next available vessel
	vessel, err := s.repo.FindAvailable(req)
	if err != nil {
		return err
	}

	// Set the vessel as part of the response message type
	res.Vessel = vessel
	return nil
}

func main() {
	vessels := []*pb.Vessel{
		&pb.Vessel{Id: "vessel001", Name: "Boaty McBoatface", MaxWeight: 200000, Capacity: 500},
	}
	repo := &VesselRepository{vessels}

	srv := micro.NewService(
		micro.Name("go.micro.srv.vessel"),
		micro.Version("latest"),
	)

	srv.Init()

	// Register our implementation with
	pb.RegisterVesselServiceHandler(srv.Server(), &service{repo})

	if err := srv.Run(); err != nil {
		fmt.Println(err)
	}
}

```

I've left a few comments, but it's pretty straight forward. Also, I'd like to note that a Reddit user /r/jerky_lodash46 pointed out that I'd used `IRepository` as my interface name previously. I'd like to correct myself here, prefixing an interface name with `I` is a convention in languages such as Java and C#, but Go doesn't really encourage this, as Go treats interfaces as first-class citizens. So I have renamed `IRepository` to `Repository`, and I've renamed my concrete struct to `ConsignmentRepository`.

In this series, I will leave in any mistakes, and correct them in future posts, so that I can explain the improvements. We can learn more that way.

Now let's get to the interesting part. When we create a consignment, we need to alter our consignment-service to call our new vessel-service, find a vessel, and update the vessel_id in the created consignment:

```go
// consignment-service/main.go
package main

import (

	// Import the generated protobuf code
	"fmt"
	"log"

	pb "github.com/EwanValentine/shippy/consignment-service/proto/consignment"
	vesselProto "github.com/EwanValentine/shippy/vessel-service/proto/vessel"
	micro "github.com/micro/go-micro"
	"golang.org/x/net/context"
)

type Repository interface {
	Create(*pb.Consignment) (*pb.Consignment, error)
	GetAll() []*pb.Consignment
}

// Repository - Dummy repository, this simulates the use of a datastore
// of some kind. We'll replace this with a real implementation later on.
type ConsignmentRepository struct {
	consignments []*pb.Consignment
}

func (repo *ConsignmentRepository) Create(consignment *pb.Consignment) (*pb.Consignment, error) {
	updated := append(repo.consignments, consignment)
	repo.consignments = updated
	return consignment, nil
}

func (repo *ConsignmentRepository) GetAll() []*pb.Consignment {
	return repo.consignments
}

// Service should implement all of the methods to satisfy the service
// we defined in our protobuf definition. You can check the interface
// in the generated code itself for the exact method signatures etc
// to give you a better idea.
type service struct {
	repo Repository
	vesselClient vesselProto.NewVesselService
}

// CreateConsignment - we created just one method on our service,
// which is a create method, which takes a context and a request as an
// argument, these are handled by the gRPC server.
func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment, res *pb.Response) error {

	// Here we call a client instance of our vessel service with our consignment weight,
	// and the amount of containers as the capacity value
	vesselResponse, err := s.vesselClient.FindAvailable(context.Background(), &vesselProto.Specification{
		MaxWeight: req.Weight,
		Capacity: int32(len(req.Containers)),
	})
	log.Printf("Found vessel: %s \n", vesselResponse.Vessel.Name)
	if err != nil {
		return err
	}

	// We set the VesselId as the vessel we got back from our
	// vessel service
	req.VesselId = vesselResponse.Vessel.Id

	// Save our consignment
	consignment, err := s.repo.Create(req)
	if err != nil {
		return err
	}

	// Return matching the `Response` message we created in our
	// protobuf definition.
	res.Created = true
	res.Consignment = consignment
	return nil
}

func (s *service) GetConsignments(ctx context.Context, req *pb.GetRequest, res *pb.Response) error {
	consignments := s.repo.GetAll()
	res.Consignments = consignments
	return nil
}

func main() {

	repo := &ConsignmentRepository{}

	// Create a new service. Optionally include some options here.
	srv := micro.NewService(

		// This name must match the package name given in your protobuf definition
		micro.Name("consignment"),
		micro.Version("latest"),
	)

	vesselClient := vesselProto.NewVesselService("go.micro.srv.vessel", srv.Client())

	// Init will parse the command line flags.
	srv.Init()

	// Register handler
	pb.RegisterShippingServiceHandler(srv.Server(), &service{repo, vesselClient})

	// Run the server
	if err := srv.Run(); err != nil {
		fmt.Println(err)
	}
}
```

Here we've created a client instance for our vessel service, this allows us to use the service name, i.e `go.micro.srv.vessel` to call the vessel service as a client and interact with its methods. In this case, just the one method (`FindAvailable`). We send our consignment weight, along with the amount of containers we want to ship as a specification to the vessel-service. Which then returns an appropriate vessel.

Update the `consignment-cli/consignment.json` file, remove the hardcoded vessel_id, we want to confirm our own is working. And let's add a few more containers and up the weight. For example:

```
{
  "description": "This is a test consignment",
  "weight": 55000,
  "containers": [
    { "customer_id": "cust001", "user_id": "user001", "origin": "Manchester, United Kingdom" },
    { "customer_id": "cust002", "user_id": "user001", "origin": "Derby, United Kingdom" },
    { "customer_id": "cust005", "user_id": "user001", "origin": "Sheffield, United Kingdom" }
  ]
}
```

Now run `$ make build && make run` in `consignment-cli`. You should see a response, with a list of created consignments. In your consignments, you should now see a vessel_id has been set.

So there we have it, two inter-connected microservices and a command line interface! The next part in the series, we will look at persisting some of this data using [MongoDB](https://www.mongodb.com/what-is-mongodb). We will also add in a third service, and use docker-compose to manage our growing ecosystem of containers locally.
