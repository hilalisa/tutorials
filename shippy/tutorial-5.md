# Microservices with Golang and Micro - Part 5

In the [previous part in this series](https://ewanvalentine.io/microservices-in-golang-part-4/), we touched upon user authentication and JWT. In this episode, we'll take a quick look at the go-micro's broker functionality and even brokering.

As mentioned in previous posts, go-micro is a pluggable framework, and it interfaces lots of different commonly used technologies. If you take a look at the [plugins repo](https://github.com/micro/go-plugins), you'll see just how many it supports out of the box.

In our case, we'll be using the NATS broker plugin.


# Event Driven Architecture
[Event driven architecture](https://en.wikipedia.org/wiki/Event-driven_architecture)' is a very simple concept at its core. We generally consider good architecture to be decoupled; that services shouldn't be coupled to, or aware of other services. When we use protocols such as gRPC, that's in some case true, because we're saying I would like to publish this request to `go.srv.user-service` for example. Which uses service discovery to find the actual location of that service. Although that doesn't directly couple us to the implementation, it does couple that service to something else called `go.srv.user-service`, so it isn't completely decoupled, as it's talking directly to something else.

So what makes event driven architecture truly decoupled? To understand this, let's first look at the process in publishing and subscribing to an event. Service a finished a task x, and then says to the ecosystem 'x has just happened'. It doesn't need to know about, or care about who is listening to that event, or what is affected by that event taking place. That responsibility is left to the listening clients.

It's also easier if you're expecting n number of services to act upon a certain event. For example, if you wanted 12 different services to act upon a new user being created using gRPC, you would have to instantiate 12 clients within your user service. With pubsub, or event driven architecture, your service doesn't need to care about that.

Now, a client service will simply listen to an event. This means that you need some kind of mediator in the middle to accept these events and inform the listening clients of their publication.

In this post, we'll create an event every time we create a user, and we will create a new service for sending out emails. We won't actually write the email implementation, just mock it out for now.


## The code
First we need to integrate the [NATS](https://nats.io/) broker plug-in into our user-service:

```go
// shippy-user-service/main.go
func main() {
    ...
    // Init will parse the command line flags.
	srv.Init()

	// Get instance of the broker using our defaults
	pubsub := srv.Server().Options().Broker

	// Register handler
	pb.RegisterUserServiceHandler(srv.Server(), &service{repo, tokenService, pubsub})
    ...
}
```

Now let's publish the event once we create a new user (see full changes [here](https://github.com/EwanValentine/shippy-user-service/tree/tutorial-5)):

```go
// shippy-user-service/handler.go
const topic = "user.created"

type service struct {
	repo         Repository
	tokenService Authable
	PubSub       broker.Broker
}
...
func (srv *service) Create(ctx context.Context, req *pb.User, res *pb.Response) error {

	// Generates a hashed version of our password
	hashedPass, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	req.Password = string(hashedPass)
	if err := srv.repo.Create(req); err != nil {
		return err
	}
	res.User = req
	if err := srv.publishEvent(req); err != nil {
		return err
	}
	return nil
}

func (srv *service) publishEvent(user *pb.User) error {
	// Marshal to JSON string
	body, err := json.Marshal(user)
	if err != nil {
		return err
	}

	// Create a broker message
	msg := &broker.Message{
		Header: map[string]string{
			"id": user.Id,
		},
		Body: body,
	}

	// Publish message to broker
	if err := srv.PubSub.Publish(topic, msg); err != nil {
		log.Printf("[pub] failed: %v", err)
	}

	return nil
}
...
```

Make sure you're running Postgres and then let's run this service:

```sh
$ docker run -d -p 5432:5432 postgres
$ make build
$ make run
```
Now let's create our email service. I've created a [new repo](https://github.com/EwanValentine/shippy-email-service) for this:

```go
// shippy-email-service
package main

import (
	"encoding/json"
	"log"

	pb "github.com/EwanValentine/shippy-user-service/proto/user"
	micro "github.com/micro/go-micro"
	"github.com/micro/go-micro/broker"
	_ "github.com/micro/go-plugins/broker/nats"
)

const topic = "user.created"

func main() {
	srv := micro.NewService(
		micro.Name("go.micro.srv.email"),
		micro.Version("latest"),
	)

	srv.Init()

	// Get the broker instance using our environment variables
	pubsub := srv.Server().Options().Broker
	if err := pubsub.Connect(); err != nil {
		log.Fatal(err)
	}

	// Subscribe to messages on the broker
	_, err := pubsub.Subscribe(topic, func(p broker.Publication) error {
		var user *pb.User
		if err := json.Unmarshal(p.Message().Body, &user); err != nil {
			return err
		}
		log.Println(user)
		go sendEmail(user)
		return nil
	})

	if err != nil {
		log.Println(err)
	}

	// Run the server
	if err := srv.Run(); err != nil {
		log.Println(err)
	}
}

func sendEmail(user *pb.User) error {
	log.Println("Sending email to:", user.Name)
	return nil
}

```
Before running this, we'll need to run [NATS](https://nats.io)...

```
$ docker run -d -p 4222:4222 nats
```

Also, I'd like to quickly explain a part of go-micro that I feel is important in understanding how it works as a framework. You will notice:

```go
srv.Init()
pubsub := srv.Server().Options().Broker
```

Let's take a quick look at that. When we create a service in go-micro, `srv.Init()` behind the scenes will look for any configuration, such as any plugins and environment variables, or command line options. It will then instantiate these integrations as part of the service. In order to then use those instances, we need to fetch them out of the service instance. Within `srv.Server().Options()`, you will also find Transport and Registry.

In our case here, it will find our `GO_MICRO_BROKER` environment variables, it will find the NATS broker plugin, and create an instance of that. Ready for us to connect to and use.

If you're creating a command line tool, you'd use `cmd.Init()`, ensuring you're importing `"github.com/micro/go-micro/cmd"`. That will have the same affect.

Now build and run this service: `$ make build && make run`, ensuring you're running the user service as well. Then head over to our shippy-user-cli repo, and run `$ make run`, watching our email service output. You should see something like... `2017/12/26 23:57:23 Sending email to: Ewan Valentine`.

And that's it! This is a bit of a tenuous example, as our email service is implicitly listening to a single 'user.created' event. But hopefully you can see how this approach allows you to write decoupled services.

It's worth mentioning that using JSON over NATS will have a performance overhead vs gRPC, as we're back into the realm of serialising json strings. But, for some use-cases, that's perfectly acceptable. NATS is incredibly efficient, and great for fire and forget events.


Go-micro also has support for some of the most widely used queue/pubsub technologies ready for you to use. [You can see a list of them here](https://github.com/micro/go-plugins/tree/master/broker). You don't have to change your implementation because go-micro abstracts that away for you. You just need to change the environment variable from `MICRO_BROKER=nats` to `MICRO_BROKER=googlepubsub`, then change the import in your main.go from `_ "github.com/micro/go-plugins/broker/nats"` to `_ "github.com/micro/go-plugins/broker/googlepubsub"` for example.

If you're not using go-micro, there's a [NATS go library](https://github.com/nats-io/go-nats)
(NATS itself is written in Go, so naturally support for Go is pretty solid).

Publishing an event:

```go
nc, _ := nats.Connect(nats.DefaultURL)

// Simple Publisher
nc.Publish("user.created", userJsonString)
```
Subscribing to an event:
```go
// Simple Async Subscriber
nc.Subscribe("user.created", func(m *nats.Msg) {
    user := convertUserString(m.Data)
    go sendEmail(user)
})
```

I mentioned earlier that when using a third party message broker, such as NATS, you lose the use of protobuf. Which is a shame, because we lose the ability to communicate using binary streams, which of course have a much lower overhead than serialised JSON strings. But like most concerns, go-micro has an answer to that problem as well.

Built-in to go-micro is a pubsub layer, which sits on top of the broker layer, but without the need for a third party broker such as NATS. But the awesome part of this feature, is that it utilises your protobuf definitions. So we're back in the realms of low-latency binary streams. So let's update our user-service to replace the existing NATS broker, with go-micro's pubsub:

```go
// shippy-user-service/main.go
func main() {
    ...
    publisher := micro.NewPublisher("user.created", srv.Client())

	// Register handler
	pb.RegisterUserServiceHandler(srv.Server(), &service{repo, tokenService, publisher})
    ...
}
```

```go
// shippy-user-service/handler.go
func (srv *service) Create(ctx context.Context, req *pb.User, res *pb.Response) error {

	// Generates a hashed version of our password
	hashedPass, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	req.Password = string(hashedPass)

    // Here's our new publisher code, much simpler
	if err := srv.repo.Create(req); err != nil {
		return err
	}
	res.User = req
	if err := srv.Publisher.Publish(ctx, req); err != nil {
		return err
	}
	return nil
}
```

Now our email service:

```go
// shippy-email-service
const topic = "user.created"

type Subscriber struct{}

func (sub *Subscriber) Process(ctx context.Context, user *pb.User) error {
	log.Println("Picked up a new message")
	log.Println("Sending email to:", user.Name)
	return nil
}

func main() {
    ...
    micro.RegisterSubscriber(topic, srv.Server(), new(Subscriber))
    ...
}
```

Now we're using our underlying User protobuf definition across our services, over gRPC, and using no third-party broker. Fantastic!

That's a wrap! Next tutorial we'll look at creating a user-interface for our services, and look at how a web client can begin to interact with our services.
