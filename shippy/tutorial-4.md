# Microservices with Golang and Micro - Part 4

In the previous part in this series, we looked at creating a user service and started storing some users. Now we need to look at making our user service store users passwords securely, and create some functionality to validate users and issue secure tokens across our microservices.

Note, I have now split out our services into separate repositories. I find this is easier to deploy. Initially I was going to attempt to do this as a monorepo, but I found it too tricky to set this up with Go's dep management without getting various conflicts. I will also begin to demonstrate how to run and test microservices independently.

Unfortunately, we'll be losing docker-compose with this approach. But that's fine for now.

You will need to run your databases manually now:
```
$ docker run -d -p 5432:5432 postgres
$ docker run -d -p 27017:27017 mongo
```

The new repositories can be found here:

- https://github.com/EwanValentine/shippy-consignment-service
- https://github.com/EwanValentine/shippy-user-service
- https://github.com/EwanValentine/shippy-vessel-service
- https://github.com/EwanValentine/shippy-user-cli
- https://github.com/EwanValentine/shippy-consignment-cli


First of all, let's update our user handler to hash our passwords, this is an absolute must. You should never, ever store plain-text passwords. Many of you will be thinking 'duh that's obvious', but unfortunately it's still done!

```go
// shippy-user-service/handler.go
...
func (srv *service) Auth(ctx context.Context, req *pb.User, res *pb.Token) error {
	log.Println("Logging in with:", req.Email, req.Password)
	user, err := srv.repo.GetByEmail(req.Email)
	log.Println(user)
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
	return nil
}
```
Not a huge amount has changed here, except we've added our password hashing functionality, and we set it as our password before saving a new user. Also, on authentication, we check against the hashed password.

Now we can securely authenticate a user against the database, we need a mechanism in which we can do this across our user interfaces and distributed services. There are many ways in which to do this, but the simplest solution I've come across, which we can use across our services and web, is [JWT](https://jwt.io).

But before we crack on, please do check the changes I've made to the Dockerfiles and the Makefiles of each service. I've also updated the imports to match the new git repositories.


## JWT
[JWT](https://jwt.io/) stands for JSON web tokens, and is a distributed security protocol. Similar to OAuth. The concept is simple, you use an algorithm to generate a unique hash for a user, which can be compared and validated against. But not only that, the token itself can contain and be made up of our users metadata. In other words, their data can itself become a part of the token. So let's look at an example of a JWT:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

The token is separated into three by .'s. Each segment has a significance. The first segment is made up of some metadata about the token itself. Such as the type of token and the algorithm used to create the token. This allows clients to understand how to decode the token. The second segment is made up of user defined metadata. This can be your users details, an expiration time, anything you wish. The final segment is the verification signature, which is information on how to hash the token and what data to use.

Of course there are also down sides and risks in using JWT, [this article](http://cryto.net/%7Ejoepie91/blog/2016/06/13/stop-using-jwt-for-sessions/) outlines those pretty well. Also, I'd recommend reading [this article](https://www.owasp.org/index.php/JSON_Web_Token_(JWT)_Cheat_Sheet_for_Java) for security best practices.

One I'd recommend you look into in particular, is getting the users origin IP, and using that to form part of the token claims. This ensures someone can't steal your token and act as you on another device. Ensuring you're using https helps to mitigate this attack type, as it obscures your token from man in the middle style attacks.

There are many different hashing algorithms you can use to hash JWT's, which commonly fall into two categories. Symmetric and Asymmetric. Symmetric is like the approach we're using, using a shared salt. Asymmetric utilises public and private keys between a client and server. This is great for authenticating across services.


Some more resources:

- [Auth0](https://auth0.com/blog/json-web-token-signing-algorithms-overview/)
- [RFC spec for algorithms](https://tools.ietf.org/html/rfc7518#section-3)

Now we've touched the on the basics of what a JWT is, let's update our `token_service.go` to perform these operations. We'll be using a fantastic Go library for this: `github.com/dgrijalva/jwt-go`, which contains some great examples.


```go
// shippy-user-service/token_service.go
package main

import (
	"time"

	pb "github.com/EwanValentine/shippy-user-service/proto/user"
	"github.com/dgrijalva/jwt-go"
)

var (

	// Define a secure key string used
	// as a salt when hashing our tokens.
	// Please make your own way more secure than this,
	// use a randomly generated md5 hash or something.
	key = []byte("mySuperSecretKeyLol")
)

// CustomClaims is our custom metadata, which will be hashed
// and sent as the second segment in our JWT
type CustomClaims struct {
	User *pb.User
	jwt.StandardClaims
}

type Authable interface {
	Decode(token string) (*CustomClaims, error)
	Encode(user *pb.User) (string, error)
}

type TokenService struct {
	repo Repository
}

// Decode a token string into a token object
func (srv *TokenService) Decode(tokenString string) (*CustomClaims, error) {

	// Parse the token
	token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(token *jwt.Token) (interface{}, error) {
		return key, nil
	})

	// Validate the token and return the custom claims
	if claims, ok := token.Claims.(*CustomClaims); ok && token.Valid {
		return claims, nil
	} else {
		return nil, err
	}
}

// Encode a claim into a JWT
func (srv *TokenService) Encode(user *pb.User) (string, error) {

	expireToken := time.Now().Add(time.Hour * 72).Unix()

	// Create the Claims
	claims := CustomClaims{
		user,
		jwt.StandardClaims{
			ExpiresAt: expireToken,
			Issuer:    "go.micro.srv.user",
		},
	}

	// Create token
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

	// Sign token and return
	return token.SignedString(key)
}

```

As per, I've left comments explaining some of the finer details, but the premise here is fairly simple. Decode takes a string token, parses it into a token object, validates it, and returns the claims if valid. This will allow us to take the user metadata from the claims in order to validate that user.

The Encode method does the opposite, this takes your custom metadata, hashes it into a new JWT and returns it.

Note we also set a 'key' variable at the top, this is a secure salt, please use something more secure than this in production.

Now we have a validate token service. Let's update our user-cli, I've simplified this to just be a script for now as I was having issues with the previous cli code, I'll come back to this, but this tool is just for testing:


```go
// shippy-user-cli/cli.go
package main

import (
	"log"
	"os"

	pb "github.com/EwanValentine/shippy-user-service/proto/user"
	micro "github.com/micro/go-micro"
	microclient "github.com/micro/go-micro/client"
	"golang.org/x/net/context"
)

func main() {

	srv := micro.NewService(

		micro.Name("go.micro.srv.user-cli"),
		micro.Version("latest"),
	)

	// Init will parse the command line flags.
	srv.Init()

	client := pb.NewUserServiceClient("go.micro.srv.user", microclient.DefaultClient)

	name := "Ewan Valentine"
	email := "ewan.valentine89@gmail.com"
	password := "test123"
	company := "BBC"

	r, err := client.Create(context.TODO(), &pb.User{
		Name:     name,
		Email:    email,
		Password: password,
		Company:  company,
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

	authResponse, err := client.Auth(context.TODO(), &pb.User{
		Email:    email,
		Password: password,
	})

	if err != nil {
		log.Fatalf("Could not authenticate user: %s error: %v\n", email, err)
	}

	log.Printf("Your access token is: %s \n", authResponse.Token)

	// let's just exit because
	os.Exit(0)
}

```

We just have some hard-coded values for now, replace those and run the script using `$ make build && make run`. You should see a token returned. Copy and paste this long token string, you will need it soon!

Now we need to update our consignment-cli to take a token string and pass it into the context to our consignment-service:

```go
// shippy-consignment-cli/cli.go
...
func main() {

	cmd.Init()

	// Create new greeter client
	client := pb.NewShippingServiceClient("go.micro.srv.consignment", microclient.DefaultClient)

	// Contact the server and print out its response.
	file := defaultFilename
	var token string
	log.Println(os.Args)

	if len(os.Args) < 3 {
		log.Fatal(errors.New("Not enough arguments, expecing file and token."))
	}

	file = os.Args[1]
	token = os.Args[2]

	consignment, err := parseFile(file)

	if err != nil {
		log.Fatalf("Could not parse file: %v", err)
	}

	// Create a new context which contains our given token.
	// This same context will be passed into both the calls we make
	// to our consignment-service.
	ctx := metadata.NewContext(context.Background(), map[string]string{
		"token": token,
	})

	// First call using our tokenised context
	r, err := client.CreateConsignment(ctx, consignment)
	if err != nil {
		log.Fatalf("Could not create: %v", err)
	}
	log.Printf("Created: %t", r.Created)

	// Second call
	getAll, err := client.GetConsignments(ctx, &pb.GetRequest{})
	if err != nil {
		log.Fatalf("Could not list consignments: %v", err)
	}
	for _, v := range getAll.Consignments {
		log.Println(v)
	}
}
```

Now we need to update our consignment-service to check the request for a token, and pass it to our user-service:

```go
// shippy-consignment-service/main.go
func main() {
    ...
    // Create a new service. Optionally include some options here.
	srv := micro.NewService(

		// This name must match the package name given in your protobuf definition
		micro.Name("go.micro.srv.consignment"),
		micro.Version("latest"),
        // Our auth middleware
		micro.WrapHandler(AuthWrapper),
	)
    ...
}

...

// AuthWrapper is a high-order function which takes a HandlerFunc
// and returns a function, which takes a context, request and response interface.
// The token is extracted from the context set in our consignment-cli, that
// token is then sent over to the user service to be validated.
// If valid, the call is passed along to the handler. If not,
// an error is returned.
func AuthWrapper(fn server.HandlerFunc) server.HandlerFunc {
	return func(ctx context.Context, req server.Request, resp interface{}) error {
		meta, ok := metadata.FromContext(ctx)
		if !ok {
			return errors.New("no auth meta-data found in request")
		}

		// Note this is now uppercase (not entirely sure why this is...)
		token := meta["Token"]
		log.Println("Authenticating with token: ", token)

		// Auth here
		authClient := userService.NewUserServiceClient("go.micro.srv.user", client.DefaultClient)
		_, err := authClient.ValidateToken(context.Background(), &userService.Token{
			Token: token,
		})
		if err != nil {
			return err
		}
		err = fn(ctx, req, resp)
		return err
	}
}
```

Now let's run our consignment-cli tool, cd into our new shippy-consignment-cli repo and run `$ make build` to build our new docker image, now run:

```
$ make build
$ docker run --net="host" \
      -e MICRO_REGISTRY=mdns \
      consignment-cli consignment.json \
      <TOKEN_HERE>
```

Notice we're using the `--net="host"` flag when running our docker containers. This tells Docker to run our containers on our host network, i.e 127.0.0.1 or localhost, rather than an internal Docker network. Note, you won't need to do any port forwarding with this approach. So instead of `-p 8080:8080` you can just do `-p 8080`. [Read more about Docker networking](https://docs.docker.com/engine/userguide/networking/).

Now when you run this, you should see a new consignment has been created. Try removing a few characters from the token, so that it becomes invalid. You should see an error.

So there we have it, we've created a JWT token service, and a middleware to validate JWT tokens to validate a user.

If you're not wanting to use go-micro and you're using vanilla grpc, you'll want your middleware to look something like:

```go
func main() {
    ...
    myServer := grpc.NewServer(
        grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(AuthInterceptor),
    )
    ...
}

func AuthInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {

    // Set up a connection to the server.
    conn, err := grpc.Dial(authAddress, grpc.WithInsecure())
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    c := pb.NewAuthClient(conn)
    r, err := c.ValidateToken(ctx, &pb.ValidateToken{Token: token})

    if err != nil {
	    log.Fatalf("could not authenticate: %v", err)
    }

    return handler(ctx, req)
}
```

This set-up's getting a little unwieldy to run locally. But we don't always need to run every service locally. We should be able to create services which are independent and can be tested in isolation. In our case, if we want to test our consignment-service, we might not necessarily want to have to run our auth-service. So one trick I use is to toggle calls to other services on or off.

I've updated our consignment-service auth wrapper:

```go
// shippy-user-service/main.go
...
func AuthWrapper(fn server.HandlerFunc) server.HandlerFunc {
	return func(ctx context.Context, req server.Request, resp interface{}) error {
        // This skips our auth check if DISABLE_AUTH is set to true
		if os.Getenv("DISABLE_AUTH") == "true" {
			return fn(ctx, req, resp)
		}
		...
	}
}
```

Then add our new toggle in our Makefile:

```
// shippy-user-service/Makefile
...
run:
	docker run -d --net="host" \
		-p 50052 \
		-e MICRO_SERVER_ADDRESS=:50052 \
		-e MICRO_REGISTRY=mdns \
		-e DISABLE_AUTH=true \
		consignment-service
```
This approach makes it easier to run certain sub-sections of your microservices locally, there are a few different approaches to this problem, but I've found this to be the easiest. I hope you've found this useful, despite the slight change in direction. Also, any advice on running go microservices as a monorepo would be greatly welcome, as it would make this series a lot easier!
