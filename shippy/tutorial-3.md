# Microservices with Golang and Micro - Part 3

In the [previous post](https://ewanvalentine.io/microservices-in-golang-part-2/), we covered some of the basics of [go-micro](https://github.com/micro/go-micro) and [Docker](https://www.docker.com/). We also introduced a second service. In this post, we're going to look at [docker-compose](https://docs.docker.com/compose/), and how we can run our services together locally a little easier. We're going to introduce some different databases, and finally we'll introduce a third service into the mix.


# Prerequisites
Install docker-compose: https://docs.docker.com/compose/install/

But first, let's look at databases.

## Choosing a database

So far our data isn't actually stored anywhere, it's stored in memory in our services, which is then lost when our containers are restarted. So of course we need a way of persisting, storing and querying our data.

The beauty of microservices, is that you can use different databases per service. Of course you don't have to do this, and many people don't. In fact I rarely do for small teams as it's a bit of a mental leap to maintain several different databases, than just one. But in some cases, one services data, might not fit the database you've used for your other services. So it makes sense to use something else. Microservices makes this trivially easy as your concerns are completely separate.

Choosing the 'correct' database for your services is an entirely different article, [this one for example](https://www.infoworld.com/article/3236291/database/how-to-choose-a-database-for-your-microservices.html), so we wont go into too much detail on this subect. However, I will say that if you have fairly loose or inconsistent datasets, then a NoSQL document store solution is perfect. They're much more flexible with what you can store and work well with json. We'll be using MongoDB for our NoSQL database. No particular reason other than it performs well, it's widely used and supported and has a great online community.

If your data is more strictly defined and relational by nature, then it can makes sense to use a traditional rdbms, or relational database. But there really aren't any hard rules, generally any will do the job. But be sure to look at your data structure, consider whether your service is doing more reading or more writing, how complex the queries will be, and try to use those as a starting point in choosing your databases. For our relational database, we'll be using Postgres. Again, no particular reason other than it does the job well and I'm familiar with it. You could use MySQL, MariaDB, or something else.

Amazon and Google both have some fantastic on premises solution for both of these database types as well, if you wanted to avoid managing your own databases (generally advisable). Another great option is [compose](https://www.compose.com/), who will spin up fully managed, scalable instances of various database technologies, using the same cloud provider as your services to avoid connection latency.

Amazon:
RDBMS: https://aws.amazon.com/rds/
NoSQL: https://aws.amazon.com/dynamodb/

Google:
RDBMS: https://cloud.google.com/spanner/
NoSQL: https://cloud.google.com/datastore/


Now that we've discussed databases a little, let's do some coding!

## docker-compose
In the last part in the series we looked at [Docker](https://docker.com), which let us run our services in light-weight containers with their own run-times and dependencies. However, it's getting slightly cumbersome to have to run and manage each service with a separate Makefile. So let's take a look at [docker-compose](https://docs.docker.com/compose/). [Docker-compose](https://docs.docker.com/compose/) allows you do define a list of docker containers in a yaml file, and specify metadata about their run-time. Docker-compose services map more or less to the same docker commands we're already using. For example:

`$ docker run -p 50052:50051 -e MICRO_SERVER_ADDRESS=:50051 -e MICRO_REGISTRY=mdns vessel-service`

Becomes:

```
version: '3.1'

services:
  vessel-service:
    build: ./vessel-service
    ports:
      - 50052:50051
    environment:
      MICRO_REGISTRY: "mdns"
      MICRO_SERVER_ADDRESS: ":50051"
```

Easy!

So let's create a docker-compose file in the root of our directory `$ touch docker-compose.yml`. Now add our services:

```
# docker-compose.yml
version: '3.1'

services:

  consignment-cli:
    build: ./consignment-cli
    environment:
      MICRO_REGISTRY: "mdns"

  consignment-service:
    build: ./consignment-service
    ports:
      - 50051:50051
    environment:
      MICRO_ADDRESS: ":50051"
      MICRO_REGISTRY: "mdns"
      DB_HOST: "datastore:27017"

  vessel-service:
    build: ./vessel-service
    ports:
      - 50052:50051
    environment:
      MICRO_ADDRESS: ":50051"
      MICRO_REGISTRY: "mdns"
```

First we define the version of docker-compose we want to use, then a list of services. There are other root level definitions such as networks and volumes, but we'll just focus on services for now.

Each service is defined by its name, then we include a `build` path, which is a reference to a location, which should contain a Dockerfile. This tells docker-compose to use this Dockerfile to build its image. You can also use `image` here to use a pre-built image. Which we will be doing later on. Then you define your port mappings, and finally your environment variables.

To build your docker-compose stack, simply run `$ docker-compose build`, and to run it, `$ docker-compose run`. To run your stack in the background, use `$ docker-compose up -d`. You can also view a list of your currently running containers at any point using `$ docker ps`. Finally, you can stop all of your current containers by running `$ docker stop $(docker ps -qa)`.

So let's run our stack. You should see lots of output and dockerfile's being built. You may also see an error from our CLI tool, but don't worry about that, it's mostly likely because it's ran prior to our other services. It's simply saying that it can't find them yet.

Let's test it all worked by running our CLI tool. To run it through docker-compose, simply run `$ docker-compose run consignment-cli` once all of the other containers are running. You should see it run successfully, just as before.


## Entities and protobufs
Throughout this series we've spoken of protobufs being the very center of our data model. We've used it to define our services structure and functionality. Because protobuf generates structs with more or less all of the correct data types, we can also re-use these structs as our underlying database models. This is actually pretty mind-blowing. It keeps in-line with the protobuf being the single source of truth.

However this approach does have its down-sides. Sometimes its tricky to marshal the code generated by protobuf into a valid database entity. Sometimes database technologies use custom types which are tricky to translate from the native types generated by protobuf. One problem I spent many many hours thinking about was how I could convert `Id string` to and from `Id bson.ObjectId` for Mongodb entities. It turns out that bson.ObjectId, is really just a string anyway, so you can marshal them together. Also, mongodb's id index is stored as `_id` internally, so you need a way to tie that to your `Id string` field as you can't really do `_Id string`. Which means finding a way to define custom tags for your protobuf files. But we'll get to that later.

Also, [many people often argue against](https://www.reddit.com/r/golang/comments/77yd72/question_do_you_use_protobufs_in_place_of_structs/) using your protobuf definitions as your database entity because you're tightly coupling your communication technology to your database code. Which is also a valid point.

Generally it's advised to convert between your protobuf definition code and your database entities. However, you end up with a lot of conversion code for converting two almost identical types, for example:

```go
func (service *Service) (ctx context.Context, req *proto.User, res *proto.Response) error {
  entity := &models.User{
    Name: req.Name.
    Email: req.Email,
    Password: req.Password,
  }
  err := service.repo.Create(entity)
  ...
}
```

Which on the surface doesn't seem all that bad, but when you've got several nested structs, and several types. It can be really tedious, and can involve a lot of iteration to convert between nested structs etc.

This approach is really down to you though, like many things in programming, this doesn't come down to a right or wrong. So take whichever approach feels most appropriate to you. But, my own personal opinion is that converting between two almost identical types, especially given we're treating our protobuf code as the basis of our data, feels like a detraction from the benefits we've attained from using protobufs as your core definition. So I will be using our protobuf code for our database. By the way, I'm not saying I'm right on this, and [I'm desperate to hear your opinions on this](mailto:ewan.valentine89@gmail.com).

Let's start hooking up our first service, our consignment service. I feel as though we should do some tidying up first. We've lumped everything into our `main.go` file. I know these are microservices, but that's no excuse to be messy! So let's create two more files in `consignment-service`, `handler.go`, `datastore.go`, and `repository.go`. I'm creating these within the root of our service, rather than creating them as new packages and directories. This is perfectly adequate for a small microservice. It's a common temptation for developers to create a structure like this:

```
main.go
models/
  user.go
handlers/
  auth.go
  user.go
services/
  auth.go
```

This harks back to the MVC days, and isn't really advised in Golang. Certainly not for smaller projects. If you had a bigger project with multiple concerns, you could organise it as followed:

```
main.go
users/
  services/
    auth.go
  handlers/
    auth.go
    user.go
  users/
    user.go
containers/
  services/
    manage.go
  models/
    container.go
```

Here you're grouping your code by domain, rather than arbitrarily grouping your code by what it does.

However, as we're dealing with a microservice, which should only really be dealing with a single concern, we don't need to take either of the above approaches. In fact, Go's ethos is to encourage simplicity. So we'll start simple and house everything in the root of our service, with some clearly defined file names.

As a side note, we'll need to update our Dockerfile's, as we're not importing our new separated code as packages, we will need to tell the go compiler to pull in these new files. So update the build function to look like this:

```
RUN CGO_ENABLED=0 GOOS=linux go build  -o consignment-service -a -installsuffix cgo main.go repository.go handler.go datastore.go
```

This will include the new files we'll be creating.


[The MongoDB Golang lib is a great example of this simplicity](https://github.com/go-mgo/mgo) and finally on this, [here's a great article on organising Go codebases](https://rakyll.org/style-packages/).

Let's start by removing all of the repository code from our main.go and re-purpose it to use the mongodb library, mgo. Once again, I've tried to comment the code to explain what each part does, so please read the code and comments thoroughly. Especially the part around how mgo handles sessions:

```go
// consignment-service/repository.go
package main

import (
	pb "github.com/EwanValentine/shippy/consignment-service/proto/consignment"
	"gopkg.in/mgo.v2"
)

const (
	dbName = "shippy"
	consignmentCollection = "consignments"
)

type Repository interface {
	Create(*pb.Consignment) error
	GetAll() ([]*pb.Consignment, error)
	Close()
}

type ConsignmentRepository struct {
	session *mgo.Session
}

// Create a new consignment
func (repo *ConsignmentRepository) Create(consignment *pb.Consignment) error {
	return repo.collection().Insert(consignment)
}

// GetAll consignments
func (repo *ConsignmentRepository) GetAll() ([]*pb.Consignment, error) {
	var consignments []*pb.Consignment
	// Find normally takes a query, but as we want everything, we can nil this.
	// We then bind our consignments variable by passing it as an argument to .All().
	// That sets consignments to the result of the find query.
	// There's also a `One()` function for single results.
	err := repo.collection().Find(nil).All(&consignments)
	return consignments, err
}

// Close closes the database session after each query has ran.
// Mgo creates a 'master' session on start-up, it's then good practice
// to copy a new session for each request that's made. This means that
// each request has its own database session. This is safer and more efficient,
// as under the hood each session has its own database socket and error handling.
// Using one main database socket means requests having to wait for that session.
// I.e this approach avoids locking and allows for requests to be processed concurrently. Nice!
// But... it does mean we need to ensure each session is closed on completion. Otherwise
// you'll likely build up loads of dud connections and hit a connection limit. Not nice!
func (repo *ConsignmentRepository) Close() {
	repo.session.Close()
}

func (repo *ConsignmentRepository) collection() *mgo.Collection {
	return repo.session.DB(dbName).C(consignmentCollection)
}
```
So there we have our code responsible for interacting with our Mongodb database. We'll need to create the code that creates the master session/connection. Update `consignment-service/datastore.go` with the following:

```go
// consignment-service/datastore.go
package main

import (
	"gopkg.in/mgo.v2"
)

// CreateSession creates the main session to our mongodb instance
func CreateSession(host string) (*mgo.Session, error) {
	session, err := mgo.Dial(host)
	if err != nil {
		return nil, err
	}

	session.SetMode(mgo.Monotonic, true)

	return session, nil
}
```

That's it, pretty straight forward. It takes a host string as an argument, returns a session to our datastore and of course a potential error, so that we can handle that on start-up. Let's modify our main.go file to hook this up to our repository:

```go
// consignment-service/main.go
package main

import (

	// Import the generated protobuf code
	"fmt"
	"log"

	pb "github.com/EwanValentine/shippy/consignment-service/proto/consignment"
	vesselProto "github.com/EwanValentine/shippy/vessel-service/proto/vessel"
	"github.com/micro/go-micro"
	"os"
)

const (
	defaultHost = "localhost:27017"
)

func main() {

	// Database host from the environment variables
	host := os.Getenv("DB_HOST")

	if host == "" {
		host = defaultHost
	}

	session, err := CreateSession(host)

	// Mgo creates a 'master' session, we need to end that session
	// before the main function closes.
	defer session.Close()

	if err != nil {

		// We're wrapping the error returned from our CreateSession
		// here to add some context to the error.
		log.Panicf("Could not connect to datastore with host %s - %v", host, err)
	}

	// Create a new service. Optionally include some options here.
	srv := micro.NewService(

		// This name must match the package name given in your protobuf definition
		micro.Name("go.micro.srv.consignment"),
		micro.Version("latest"),
	)

	vesselClient := vesselProto.NewVesselServiceClient("go.micro.srv.vessel", srv.Client())

	// Init will parse the command line flags.
	srv.Init()

	// Register handler
	pb.RegisterShippingServiceHandler(srv.Server(), &service{session, vesselClient})

	// Run the server
	if err := srv.Run(); err != nil {
		fmt.Println(err)
	}
}
```

## Copy vs Clone
You may have noticed that when using the mgo Mongodb library. We create a database session, which is passed into our handlers, but on each request we call a method which clones that session and passes it into our repository code.

Effectively, aside from spawning the first connection to the database, we never touch the 'master session', we called `session.Clone()` each time we want to make a call to the datastore. As I mentioned briefly in the code comments, but I think it's worth re-iterating this in some detail, if you use the master session, you are re-using the same socket. Which means your queries may become blocked by other queries and have to wait for operations to finish on this socket. Which is pointless in a language which supports concurrency.

So to avoid blocking requests, mgo allows you to `Copy()` or `Clone()` a session, so that you have a concurrent connection for each request. You will notice that I've mentioned `Copy` and `Clone` methods, these are very similar, but have a subtle, but important difference. Clone re-uses the same socket as master. Which reduces the overhead of spawning an entirely new socket. Which is perfect for fast write performance. However, longer operations, such as more complex queries or big data jobs etc, may cause blocking in other go routines attempting to use this socket.

Generally speaking, you're probably better off with Clone for purposes such as ours.


The final bit of tidying up we need to do is to move our gRPC handler code out into our new `handler.go` file. So let's do that.

```go
// consignment-service.go

package main

import (
	"log"
	"golang.org/x/net/context"
	pb "github.com/EwanValentine/shippy/consignment-service/proto/consignment"
	vesselProto "github.com/EwanValentine/shippy/vessel-service/proto/vessel"
)

// Service should implement all of the methods to satisfy the service
// we defined in our protobuf definition. You can check the interface
// in the generated code itself for the exact method signatures etc
// to give you a better idea.
type service struct {
	vesselClient vesselProto.VesselServiceClient
}

func (s *service) GetRepo() Repository {
    return &ConsignmentRepository{s.session.Clone()}
}

// CreateConsignment - we created just one method on our service,
// which is a create method, which takes a context and a request as an
// argument, these are handled by the gRPC server.
func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment, res *pb.Response) error {
    repo := s.GetRepo()
    defer repo.Close()
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
	err = repo.Create(req)
	if err != nil {
		return err
	}

	// Return matching the `Response` message we created in our
	// protobuf definition.
	res.Created = true
	res.Consignment = req
	return nil
}

func (s *service) GetConsignments(ctx context.Context, req *pb.GetRequest, res *pb.Response) error {
    repo := s.GetRepo()
    defer repo.Close()
	consignments, err := repo.GetAll()
	if err != nil {
		return err
	}
	res.Consignments = consignments
	return nil
}

```

We've updated some of the return arguments in our repo slightly from the last tutorial:
Old:
```go
type Repository interface {
	Create(*pb.Consignment) (*pb.Consignment, error)
	GetAll() []*pb.Consignment
}
```

New:
```go
type Repository interface {
	Create(*pb.Consignment) error
	GetAll() ([]*pb.Consignment, error)
    Close()
}
```

This is just because I felt we didn't need to return the same consignment after creating it. And now we're returning a proper error from mgo for our get query. Otherwise the code is more or less the same. Oh and of course we add our `Close` method to the interface.

Now let's do the same to your vessel-service. I'm not going to demonstrate this in this post, you should have a good feel for it yourself at this point. Remember you can use [my repository](https://github.com/EwanValentine/shippy/tree/tutorial-3) as a reference.

 We will however add a new method to our vessel-service, which will allow us to create new vessels. As ever, let's start by updating our protobuf definition:

```
syntax = "proto3";

package vessel;

service VesselService {
  rpc FindAvailable(Specification) returns (Response) {}
  rpc Create(Vessel) returns (Response) {}
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
  bool created = 3;
}
```
We created a new `Create` method under our gRPC service, which takes a vessel and returns our generic response. We've added a new field to our response message as well, just a `created` bool. Run `$ make build` to update this service. Now we'll add a new handler in `vessel-service/handler.go` and a new repository method:

```go
// vessel-service/handler.go

func (s *service) GetRepo() Repository {
    return &VesselRepository{s.session.Clone()}
}

func (s *service) Create(ctx context.Context, req *pb.Vessel, res *pb.Response) error {
    repo := s.GetRepo()
    defer repo.Close()
	if err := repo.Create(req); err != nil {
		return err
	}
	res.Vessel = req
	res.Created = true
	return nil
}
```

```go
// vessel-service/repository.go
func (repo *VesselRepository) Create(vessel *pb.Vessel) error {
	return repo.collection().Insert(vessel)
}
```

Now we can create vessels! I've update the main.go to use our new Create method to store our dummy data, [see here](https://github.com/EwanValentine/shippy/blob/master/vessel-service/main.go).

So after all of that. We have updated our services to use Mongodb. Before we try to run this, we will need to update our `docker-compose` file to include a Mongodb container:

```
services:
  ...
  datastore:
    image: mongo
    ports:
      - 27017:27017

```
And update the environment variables in your two services to include: `DB_HOST: "datastore:27017"`. Notice, we're calling `datastore` as our host name, and not `localhost` for example. This is because docker-compose handles some clever internal DNS stuff for us.

So you should have:

```
version: '3.1'

services:

  consignment-cli:
    build: ./consignment-cli
    environment:
      MICRO_REGISTRY: "mdns"

  consignment-service:
    build: ./consignment-service
    ports:
      - 50051:50051
    environment:
      MICRO_ADDRESS: ":50051"
      MICRO_REGISTRY: "mdns"
      DB_HOST: "datastore:27017"

  vessel-service:
    build: ./vessel-service
    ports:
      - 50052:50051
    environment:
      MICRO_ADDRESS: ":50051"
      MICRO_REGISTRY: "mdns"
      DB_HOST: "datastore:27017"

  datastore:
    image: mongo
    ports:
      - 27017:27017
```

Re-build your stack `$ docker-compose build` and re-run it `$ docker-compose up`. Note, sometimes because of Dockers caching, you may need to run a cacheless build to pick up certain changes. To do this in docker-compose, simply use the `--no-cache` flag when running `$ docker-compose build`.


## User service

Now let's create a third service. We'll start by updating our `docker-compose.yml` file. Also, to mix things up a bit, we'll add Postgres to our docker stack for our user service:

```
  ...
  user-service:
    build: ./user-service
    ports:
      - 50053:50051
    environment:
      MICRO_ADDRESS: ":50051"
      MICRO_REGISTRY: "mdns"

  ...
  database:
    image: postgres
    ports:
      - 5432:5432
```

Now create a `user-service` directory in your root. And, as per the previous services. Create the following files: handler.go, main.go, repository.go, database.go, Dockerfile, Makefile, a sub-directory for our proto files, and finally the proto file itself: `proto/user/user.proto`.

Add the following to `user.proto`:

```
syntax = "proto3";

package go.micro.srv.user;

service UserService {
    rpc Create(User) returns (Response) {}
    rpc Get(User) returns (Response) {}
    rpc GetAll(Request) returns (Response) {}
    rpc Auth(User) returns (Token) {}
    rpc ValidateToken(Token) returns (Token) {}
}

message User {
    string id = 1;
    string name = 2;
    string company = 3;
    string email = 4;
    string password = 5;
}

message Request {}

message Response {
    User user = 1;
    repeated User users = 2;
    repeated Error errors = 3;
}

message Token {
    string token = 1;
    bool valid = 2;
    repeated Error errors = 3;
}

message Error {
    int32 code = 1;
    string description = 2;
}
```  

Now, ensuring you've created a Makefile similar to that of our previous services, you should be able to run `$ make build` to generate our gRPC code. As per our previous services, we've created some code to interface our gRPC methods. We're only going to make a few of them work in this part of the series. We just want to be able to create and fetch a user. In the next part of the series, we'll be looking at authentication and JWT. So we'll be leaving anything token related for now. Your handlers should look like this:

```go
// user-service/handler.go
package main

import (
	"golang.org/x/net/context"
	pb "github.com/EwanValentine/shippy/user-service/proto/user"
)

type service struct {
	repo Repository
	tokenService Authable
}

func (srv *service) Get(ctx context.Context, req *pb.User, res *pb.Response) error {
	user, err := srv.repo.Get(req.Id)
	if err != nil {
		return err
	}
	res.User = user
	return nil
}

func (srv *service) GetAll(ctx context.Context, req *pb.Request, res *pb.Response) error {
	users, err := srv.repo.GetAll()
	if err != nil {
		return err
	}
	res.Users = users
	return nil
}

func (srv *service) Auth(ctx context.Context, req *pb.User, res *pb.Token) error {
	user, err := srv.repo.GetByEmailAndPassword(req)
	if err != nil {
		return err
	}
	res.Token = "testingabc"
	return nil
}

func (srv *service) Create(ctx context.Context, req *pb.User, res *pb.Response) error {
	if err := srv.repo.Create(req); err != nil {
		return err
	}
	res.User = req
	return nil
}

func (srv *service) ValidateToken(ctx context.Context, req *pb.Token, res *pb.Token) error {
	return nil
}
```

Now let's add our repository code:

```go
// user-service/repository.go
package main

import (
	pb "github.com/EwanValentine/shippy/user-service/proto/user"
	"github.com/jinzhu/gorm"
)

type Repository interface {
	GetAll() ([]*pb.User, error)
	Get(id string) (*pb.User, error)
	Create(user *pb.User) error
	GetByEmailAndPassword(user *pb.User) (*pb.User, error)
}

type UserRepository struct {
	db *gorm.DB
}

func (repo *UserRepository) GetAll() ([]*pb.User, error) {
	var users []*pb.User
	if err := repo.db.Find(&users).Error; err != nil {
		return nil, err
	}
	return users, nil
}

func (repo *UserRepository) Get(id string) (*pb.User, error) {
	var user *pb.User
	user.Id = id
	if err := repo.db.First(&user).Error; err != nil {
		return nil, err
	}
	return user, nil
}

func (repo *UserRepository) GetByEmailAndPassword(user *pb.User) (*pb.User, error) {
	if err := repo.db.First(&user).Error; err != nil {
		return nil, err
	}
	return user, nil
}

func (repo *UserRepository) Create(user *pb.User) error {
	if err := repo.db.Create(user).Error; err != nil {
		return err
	}
}
```

We also need to change our ORM's behaviour to generate a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) on creation, instead of trying to generate an integer ID. In case you didn't know, a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) is a randomly generated set of hyphenated strings, used as an ID or primary key. This is more secure than just using auto-incrementing ID's, because it stops people from guessing or traversing through your API endpoints. MongoDB already uses a variation of this, but we need to tell our Postgres models to use UUID's. So in `user-service/proto/user` create a new file called `extensions.go`, in that file, add:

```go
package go_micro_srv_user

import (
	"github.com/jinzhu/gorm"
	"github.com/satori/go.uuid"
)

func (model *User) BeforeCreate(scope *gorm.Scope) error {
	uuid := uuid.NewV4()
	return scope.SetColumn("Id", uuid.String())
}
```

This hooks into GORM's [event lifecycle](http://jinzhu.me/gorm/callbacks.html) so that we generate a UUID for our Id column, before the entity is saved.

You'll notice here, unlike our Mongodb services, we're not doing any connection handling. The native, SQL/postgres drivers work slightly differently, so we don't need to worry about that this time. We're using a package called 'gorm', let's touch on this briefly.

## Gorm - Go + ORM

[Gorm](http://jinzhu.me/gorm/) is a reasonably light-weight object relational mapper, which works nicely with Postgres, MySQL, Sqlite etc. It's very easy to set-up, use and manages your database schema changes automatically.

That being said, with microservices, your data structures are much smaller, contain less joins and overall complexity. So don't feel as though you should use an ORM of any kind.

We need to be able to test creating a user, so let's create another cli tool. This time `user-cli` in our project root. Similar as our consignment-cli, but this time:

```go
package main

import (
	"log"
	"os"

	pb "github.com/EwanValentine/shippy/user-service/proto/user"
	microclient "github.com/micro/go-micro/client"
	"github.com/micro/go-micro/cmd"
	"golang.org/x/net/context"
	"github.com/micro/cli"
	"github.com/micro/go-micro"
)


func main() {

	cmd.Init()

	// Create new greeter client
	client := pb.NewUserServiceClient("go.micro.srv.user", microclient.DefaultClient)

    // Define our flags
	service := micro.NewService(
		micro.Flags(
			cli.StringFlag{
				Name:  "name",
				Usage: "You full name",
			},
			cli.StringFlag{
				Name:  "email",
				Usage: "Your email",
			},
			cli.StringFlag{
				Name:  "password",
				Usage: "Your password",
			},
			cli.StringFlag{
				Name: "company",
				Usage: "Your company",
			},
		),
	)

    // Start as service
	service.Init(

		micro.Action(func(c *cli.Context) {

			name := c.String("name")
			email := c.String("email")
			password := c.String("password")
			company := c.String("company")

            // Call our user service
			r, err := client.Create(context.TODO(), &pb.User{
				Name: name,
				Email: email,
				Password: password,
				Company: company,
			})
			if err != nil {
				log.Fatalf("Could not create: %v", err)
			}
			log.Printf("Created: %s", r.User.Id)

			getAll, err := client.GetAll(context.Background(), &pb.Request{})
			if err != nil {
				log.Fatalf("Could not list users: %v", err)
			}
			for _, v := range getAll.Users {
				log.Println(v)
			}

			os.Exit(0)
		}),
	)

	// Run the server
	if err := service.Run(); err != nil {
		log.Println(err)
	}
}

```

Here we've used go-micro's command line helper, which is really neat.


We can run this and create a user:

```
$ docker-compose run user-cli command \
  --name="Ewan Valentine" \
  --email="ewan.valentine89@gmail.com" \
  --password="Testing123" \
  --company="BBC"
```

And you should see the created user in a list!

This isn't very secure, as currently we're storing plain-text passwords, but in the next part of the series, we'll be looking at authentication and JWT tokens across our services.

So there we have it, we've created an additional service, an additional command line tool, and we've started to persist our data using two different database technologies. We've covered a lot of ground in this post, and apologies if we went over anything too quickly, covered too much or assumed too much knowledge.
