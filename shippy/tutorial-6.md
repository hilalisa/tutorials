# Microservices with Golang and Micro - Part 6

[In the previous post](http://ewanvalentine.io/microservices-in-golang-part-5) we looked at some of the various approaches to event-driven architecture in go-micro and in go in general. This part, we're going to delve into the client side and take a look at how we can create web clients which interact with our platform.

We'll take a look at how [micro](https://github.com/micro/micro) toolkit, enables you to proxy your internal rpc methods externally to web clients.

We'll be creating a user interfaces for our platform, a login screen, and a create consignment interface. This will tie together some of the previous posts.

So let's begin!


## The RPC renaissance
REST has served the web well for many years now, and it has rapidly become the goto way of managing resources between clients and servers. REST came about to replace a wild west of RPC and SOAP implementations, which felt dated and painful at times. Ever had to write a wsdl file?

REST promised us a pragmatic, simple, and unified approach to managing resources. REST used http verbs to be more explicit in the type of action being performed. REST encouraged us to use http error codes to better describe the response from the server. And for the most part, this approach worked well, and was fine. But like all good things, REST had many gripes and annoyances, that I'm not going to detail in full here. But do give [this article](https://medium.freecodecamp.org/rest-is-the-new-soap-97ff6c09896d) a read. But RPC is making a comeback with the advent of microservices.

Rest is great for managing different resources, but a microservice typically only deals with a single resource by its very nature. So we don't need to use RESTful terminology in the context of microservices. Instead we can focus on specific actions and interactions with each service.


## Micro
We've been using go-micro extensively throughout this tutorial, we'll now touch on the micro cli/toolkit. The micro toolkit provides an API gateway, a sidecar, a web proxy, and a few other cool features. But the main part we'll be looking at in this post is the API gateway.

The API gateway will allow us to proxy our rpc calls to web friendly JSON rpc calls, and will expose urls which we can use in our client side applications.

So how does this work? You firstly ensure that you have the micro toolkit installed:
```
$ go get -u github.com/micro/micro
```

Better still, since we're using Docker, let's use the docker image:
```
$ docker pull microhq/micro
```

Now let's head into our user service, I made a few changes, mostly error handling and naming conventions:

```go
// shippy-user-service/main.go
package main

import (
	"log"

	pb "github.com/EwanValentine/shippy-user-service/proto/auth"
	"github.com/micro/go-micro"
	_ "github.com/micro/go-plugins/registry/mdns"
)

func main() {

	// Creates a database connection and handles
	// closing it again before exit.
	db, err := CreateConnection()
	defer db.Close()

	if err != nil {
		log.Fatalf("Could not connect to DB: %v", err)
	}

	// Automatically migrates the user struct
	// into database columns/types etc. This will
	// check for changes and migrate them each time
	// this service is restarted.
	db.AutoMigrate(&pb.User{})

	repo := &UserRepository{db}

	tokenService := &TokenService{repo}

	// Create a new service. Optionally include some options here.
	srv := micro.NewService(

		// This name must match the package name given in your protobuf definition
		micro.Name("shippy.auth"),
	)

	// Init will parse the command line flags.
	srv.Init()

    // Will comment this out for now to save having to run this locally...
	// publisher := micro.NewPublisher("user.created", srv.Client())

	// Register handler
	pb.RegisterAuthHandler(srv.Server(), &service{repo, tokenService, publisher})

	// Run the server
	if err := srv.Run(); err != nil {
		log.Fatal(err)
	}
}

```

```
// shippy-user-service/proto/auth/auth.proto
syntax = "proto3";

package auth;

service Auth {
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

And...

```go
// shippy-user-service/handler.go
package main

import (
	"errors"
	"fmt"
	"log"

	pb "github.com/EwanValentine/shippy-user-service/proto/auth"
	micro "github.com/micro/go-micro"
	"golang.org/x/crypto/bcrypt"
	"golang.org/x/net/context"
)

const topic = "user.created"

type service struct {
	repo         Repository
	tokenService Authable
	Publisher    micro.Publisher
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
	log.Println("Logging in with:", req.Email, req.Password)
	user, err := srv.repo.GetByEmail(req.Email)
	log.Println(user, err)
	if err != nil {
		return err
	}

	// Compares our given password against the hashed password
	// stored in the database
	if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
		return err
	}

	token, err := srv.tokenService.Encode(user)
	if err != nil {
		return err
	}
	res.Token = token
	return nil
}

func (srv *service) Create(ctx context.Context, req *pb.User, res *pb.Response) error {

	log.Println("Creating user: ", req)

	// Generates a hashed version of our password
	hashedPass, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		return errors.New(fmt.Sprintf("error hashing password: %v", err))
	}

	req.Password = string(hashedPass)
	if err := srv.repo.Create(req); err != nil {
		return errors.New(fmt.Sprintf("error creating user: %v", err))
	}

	res.User = req
	if err := srv.Publisher.Publish(ctx, req); err != nil {
		return errors.New(fmt.Sprintf("error publishing event: %v", err))
	}

	return nil
}

func (srv *service) ValidateToken(ctx context.Context, req *pb.Token, res *pb.Token) error {

	// Decode token
	claims, err := srv.tokenService.Decode(req.Token)

	if err != nil {
		return err
	}

	if claims.User.Id == "" {
		return errors.New("invalid user")
	}

	res.Valid = true

	return nil
}

```

Now run `$ make build && make run`. Then head over to your shippy-email-service and run `$ make build && make run`. Once both of these are running, run:

```
$ docker run -p 8080:8080 \
    -e MICRO_REGISTRY=mdns \
    microhq/micro api \
    --handler=rpc \
    --address=:8080 \
    --namespace=shippy
```

This runs the micro api-gateway as an rpc handler on port 8080 in a Docker container. We're telling it to use mdns as our registry locally, the same as our other services. Finally, we're telling it to use the namespace `shippy`, this is the first part of all of our service names. I.e `shippy.auth` or `shippy.email`. It's important to set this as it defaults to `go.micro.api`, in which case, it won't be able to find our services in order to proxy them.

Our user service methods can now be called using the following:

Create a user:
```
curl -XPOST -H 'Content-Type: application/json' \
    -d '{ "service": "shippy.auth", "method": "Auth.Create", "request": { "user": { "email": "ewan.valentine89@gmail.com", "password": "testing123", "name": "Ewan Valentine", "company": "BBC" } } }' \
    http://localhost:8080/rpc
```

As you can see, we include in our request, the service we want to be routed to, the method on that service we want to use, and finally the data we wish to be used.


Authenticate a user:
```
$ curl -XPOST -H 'Content-Type: application/json' \
    -d '{ "service": "shippy.auth", "method": "Auth.Auth", "request":  { "email": "your@email.com", "password": "SomePass" } }' \
    http://localhost:8080/rpc
```

Great!


## Consignment service

Now it's time to fire up our consignment service again, `$ make build && make run`. We shouldn't need to change anything here, but, running the rpc proxy, we should be able to do:

Create a consignment:
```
$ curl -XPOST -H 'Content-Type: application/json' \
    -d '{
      "service": "shippy.consignment",
      "method": "ConsignmentService.Create",
      "request": {
        "description": "This is a test",
        "weight": "500",
        "containers": []
      }
    }' --url http://localhost:8080/rpc
```

## Vessel service
The final service we will need to run in order to test our user interface, is our vessel service, nothing's changed here eiter, so just run `$ make build && make run`.


## User interface

Let's make use of our new rpc endpoints, and create a UI. We're going to be using React, but feel free to use whatever you like. The requests are all the same. I'm going to be using the `react-create-app` library from Facebook. `$ npm install -g react-create-app`. Once that's installed you can do `$ react-create-app shippy-ui`. And that will create the skeleton of a React application for you.

So let's begin...

```javascript
// shippy-ui/src/App.js
import React, { Component } from 'react';
import './App.css';
import CreateConsignment from './CreateConsignment';
import Authenticate from './Authenticate';

class App extends Component {

  state = {
    err: null,
    authenticated: false,
  }

  onAuth = (token) => {
    this.setState({
      authenticated: true,
    });
  }

  renderLogin = () => {
    return (
      <Authenticate onAuth={this.onAuth} />
    );
  }

  renderAuthenticated = () => {
    return (
      <CreateConsignment />
    );
  }

  getToken = () => {
    return localStorage.getItem('token') || false;
  }

  isAuthenticated = () => {
    return this.state.authenticated || this.getToken() || false;
  }

  render() {
    const authenticated = this.isAuthenticated();
    return (
      <div className="App">
        <div className="App-header">
          <h2>Shippy</h2>
        </div>
        <div className='App-intro container'>
          {(authenticated ? this.renderAuthenticated() : this.renderLogin())}
        </div>
      </div>
    );
  }
}

export default App;

```

Now let's add our main two components, Authenticate and CreateConsignment:

```javascript
// shippy-ui/src/Authenticate.js
import React from 'react';

class Authenticate extends React.Component {

  constructor(props) {
    super(props);
  }

  state = {
    authenticated: false,
    email: '',
    password: '',
    err: '',
  }

  login = () => {
    fetch(`http://localhost:8080/rpc`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        request: {
          email: this.state.email,
          password: this.state.password,
        },
        service: 'shippy.auth',
        method: 'Auth.Auth',
      }),
    })
    .then(res => res.json())
    .then(res => {
      this.props.onAuth(res.token);
      this.setState({
        token: res.token,
        authenticated: true,
      });
    })
    .catch(err => this.setState({ err, authenticated: false, }));
  }

  signup = () => {
    fetch(`http://localhost:8080/rpc`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        request: {
          email: this.state.email,
          password: this.state.password,
          name: this.state.name,
        },
        method: 'Auth.Create',
        service: 'shippy.auth',
      }),
    })
    .then((res) => res.json())
    .then((res) => {
      this.props.onAuth(res.token.token);
      this.setState({
        token: res.token.token,
        authenticated: true,
      });
      localStorage.setItem('token', res.token.token);
    })
    .catch(err => this.setState({ err, authenticated: false, }));
  }

  setEmail = e => {
    this.setState({
      email: e.target.value,
    });
  }

  setPassword = e => {
    this.setState({
      password: e.target.value,
    });
  }

  setName = e => {
    this.setState({
      name: e.target.value,
    });
  }

  render() {
    return (
      <div className='Authenticate'>
        <div className='Login'>
          <div className='form-group'>
            <input
              type="email"
              onChange={this.setEmail}
              placeholder='E-Mail'
              className='form-control' />
          </div>
          <div className='form-group'>
            <input
              type="password"
              onChange={this.setPassword}
              placeholder='Password'
              className='form-control' />
          </div>
          <button className='btn btn-primary' onClick={this.login}>Login</button>
          <br /><br />
        </div>
        <div className='Sign-up'>
          <div className='form-group'>
            <input
              type='input'
              onChange={this.setName}
              placeholder='Name'
              className='form-control' />
          </div>
          <div className='form-group'>
            <input
              type='email'
              onChange={this.setEmail}
              placeholder='E-Mail'
              className='form-control' />
          </div>
          <div className='form-group'>
            <input
              type='password'
              onChange={this.setPassword}
              placeholder='Password'
              className='form-control' />
          </div>
          <button className='btn btn-primary' onClick={this.signup}>Sign-up</button>
        </div>
      </div>
    );
  }
}

export default Authenticate;

```

and...

```javascript
// shippy-ui/src/CreateConsignment.js
import React from 'react';
import _ from 'lodash';

class CreateConsignment extends React.Component {

  constructor(props) {
    super(props);
  }

  state = {
    created: false,
    description: '',
    weight: 0,
    containers: [],
    consignments: [],
  }

  componentWillMount() {
    fetch(`http://localhost:8080/rpc`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        service: 'shippy.consignment',
        method: 'ConsignmentService.Get',
        request: {},
      })
    })
    .then(req => req.json())
    .then((res) => {
      this.setState({
        consignments: res.consignments,
      });
    });
  }

  create = () => {
    const consignment = this.state;
    fetch(`http://localhost:8080/rpc`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        service: 'shippy.consignment',
        method: 'ConsignmentService.Create',
        request: _.omit(consignment, 'created', 'consignments'),
      }),
    })
    .then((res) => res.json())
    .then((res) => {
      this.setState({
        created: res.created,
        consignments: [...this.state.consignments, consignment],
      });
    });
  }

  addContainer = e => {
    this.setState({
      containers: [...this.state.containers, e.target.value],
    });
  }

  setDescription = e => {
    this.setState({
      description: e.target.value,
    });
  }

  setWeight = e => {
    this.setState({
      weight: Number(e.target.value),
    });
  }

  render() {
    const { consignments, } = this.state;
    return (
      <div className='consignment-screen'>
        <div className='consignment-form container'>
          <br />
          <div className='form-group'>
            <textarea onChange={this.setDescription} className='form-control' placeholder='Description'></textarea>
          </div>
          <div className='form-group'>
            <input onChange={this.setWeight} type='number' placeholder='Weight' className='form-control' />
          </div>
          <div className='form-control'>
            Add containers...
          </div>
          <br />
          <button onClick={this.create} className='btn btn-primary'>Create</button>
          <br />
          <hr />
        </div>
        {(consignments && consignments.length > 0
          ? <div className='consignment-list'>
              <h2>Consignments</h2>
              {consignments.map((item) => (
                <div>
                  <p>Vessel id: {item.vessel_id}</p>
                  <p>Consignment id: {item.id}</p>
                  <p>Description: {item.description}</p>
                  <p>Weight: {item.weight}</p>
                  <hr />
                </div>
              ))}
            </div>
          : false)}
      </div>
    );
  }
}

export default CreateConsignment;

```

*Note: I also added Twitter Bootstrap into /public/index.html and altered some of the css*.

Now run the user interface `$ npm start`. This should automatically open within your browser. You should now be able to sign-up and log-in and view a consignment form where you can create new consignments. Take a look in the network tab in your dev tools, and take a look at the rpc methods firing and fetching our data from our different microservices.
