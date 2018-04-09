# Microservices with Golang and Micro - Part 8

In the [previous post](https://ewanvalentine.io/microservices-in-golang-part-7/) we looked at creating a container engine cluster with [Terraform](https://terraform.io). In this post, we'll look at deploying containers into our cluster using Container Engine and [Kubernetes](https://kubernetes.io/).


## Kubernetes
Firstly, what is [Kubernetes](https://kubernetes.io/)? [Kubernetes](https://kubernetes.io/) is an open-source, container management framework. It is platform agnostic, meaning you can run this on your local machine, on AWS, on Google Cloud, or anywhere else. It gives you control over groups of containers and their network rules, using declerative configuration.

You simply write yaml/json files which describe which containers should be running, and where. You define your networking rules, such as any port forwarding. It also handles service discovery for you.

Kubernetes is one of the most important additions to the cloud scene, and is quickly becoming the defacto choice for cloud container management. So this is a good one to understand.

So let's get started!

Firsly, ensure you have the kubectl cli installed locally:
```bash
$ gcloud components install kubectl
```

Now let's ensure you're connected to your cluster and correctly authenticated. Firsty, we'll log in and ensure we're authenticated. Secondly we'll set the project settings to ensure we're using the correct project ID and availability zone.

```bash
$ echo "This command will open a web browser, and will ask you to login
$ gcloud auth application-default login

$ gcloud config set project shippy-freight
$ gcloud config set compute/zone eu-west2-a

$ echo "Now generate a security token and access to your KB cluster"
$ gcloud container clusters get-credentials shippy-freight-cluster
```


In the above commands, you may need to replace the compute/zone with whichever region you've chosen, your project id and cluster name's may also differ to mine.

Here's an outline...

```bash
$ echo "This command will open a web browser, and will ask you to login
$ gcloud auth application-default login

$ gcloud config set project <project-id>
$ gcloud config set compute/zone <availability-zone>

$ echo "Now generate a security token and access to your KB cluster"
$ gcloud container clusters get-credentials <cluster-name>
```

Your project ID can be found by clicking here...

![Screen-Shot-2018-03-17-at-17.55.41](/content/images/2018/03/Screen-Shot-2018-03-17-at-17.55.41.png)

Now look for your project ID...

![Screen-Shot-2018-03-17-at-17.56.35](/content/images/2018/03/Screen-Shot-2018-03-17-at-17.56.35.png)

Your clusters region/zone and cluster name can be found by clicking the 'Compute Engine' in the left side menu, then click 'VM Instances'. You'll see your Kubernetes VM, click into that for more details, and you should be able to find ever piece of information regarding your cluster here.


So now if you run...

```bash
$ kubectl get pods
```

You should see... `No resources found.`. That's fine, we haven't deployed anything yet. So let's start thinking about what it is we actually need to deploy. We need a Mongodb instance. Typically, you'd deploy a mongodb instance, or database instance along with every service, for complete separation. But for this example, we're going to cheat and just stick with one centralised instance. This is a single point of failure however, so in a real-world scenario, you might want to consider splitting out your database instances more in line with your services. However, our way is also okay.

Then we need to deploy our services, vessel-service, user-service, consignment-service and email-service. Okay, that's easy enough!

Let's start with our Mongodb instance. As this doesn't belong to a single service, and this is part of the platform as a whole, we'll keep these deployments in our shippy-infrastructure repo. Which I've ommitted from Github as it contains too much sensitive data, but I'll give you my full deployment files.

First we need a config to create an ssd, for long-term storage. This is so that when our containers restart, they don't lose all of their data.

```yaml
// shippy-infrastructure/deployments/mongodb-ssd.yml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

Now our deployment file (we'll go into these in a little more detail throughout the post)...
```yaml
// shippy-infrastructure/deployments/mongodb-deployment.yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
        role: mongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--smallfiles"
            - "--noprealloc"
            - "--bind_ip"
            - "0.0.0.0"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=test"
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi

```

And a service file...

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
```

A lot of that, at this point might be meaningless to you. So let's try and clear some of the key concepts up in Kubernetes.


## Nodes
[Read the docs](https://kubernetes.io/docs/concepts/architecture/nodes/)
Nodes are your VMs or physical servers, your containers are clustered across your nodes, and services are used to expose access between the groups of containers running on each node/pod.

## Pods
[Read the docs](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
Pods are groups of related containers. For instance, a pod could contain your auth-service container, your user database container, your login signup user interface etc. These containers are all clearly related. Pods allow you to group them together so that they always have access to one another, are running in the same immediate network space, and allow you to scale them as a group. This is really cool! Pods are a very misunderstood feature in Kubernetes.

## Deployments
[Read the docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

Deployments are state controllers, a deployment is a description of what the final outcome should be, and what it should be kept at. A deployment is an instruction to Kubernetes, saying for example, I would like three of these containers, running on these ports, with these environment variables. Kubernetes will ensure that that state is maintained. If one of the containers crashes, and goes down to two containers, it will bring up another to meet the demand for three containers.

## StatefulSet
[Read the docs](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

A stateful set is very similar to a deployment, except it will retain, using some kind of storage, some state regarding the containers. This is ideal for sharded datstores for example.

Under the hood, Mongodb writes data to a binary datastore format, most databases do something of this nature. In creating a database instance within something disposible, such as a docker container. Your data will be lost if the container restarts. Traditionally you would use volumes to mount the data/files on container start.

You can do this with deployments in Kubernetes. But StatefulSets, has some additional automation around this with regards to clustering nodes. So it's a more natural fit for our mongodb containers.


## Services
[Read the docs](https://kubernetes.io/docs/concepts/services-networking/service/)

A service is a grouping of network rules, such as port forwarding and DNS rules; which connects your pods together at a network level, and controls who can talk to who, or what can be accessed from the outside world.

There are two main types of service that you'll likely come across, a load balancer and a node port.

A load balancer, is a round-robbin load-balancer, which creates an IP address to proxy against your selected node. You would use this to expose a service out to the public.

A node port exposes pods to the top-level network space, so that they can be accesses by other services, internally to other pods/instances. These are useful for exposing nodes out to other pods. This is the one you'd use to allow services to communicate with one another. This is your service discovery in essence. Or at least a part of it.

This has been a very whistle-stop tour of Kubernetes, there's a lot more to it, keep digging and reading. It's worth noting also that if you're a docker user on your local computer, if you're using edge version of docker for mac/windows for example, you can spin up a Kubernetes cluster locally on your machine. This is really useful for testing.

So we've created three files, one for storage, one for our stateful set, and one for our service. The outcome of this will be a replicated set of mongodb containers, with stateful storage and a service exposing the datastore across our other pods. Let's go ahead and create these, be sure to do this in the correct order, as some depend on each other.


```bash
echo "shippy-infrastructure"
$ kubectl create -f ./deployments/mongodb-ssd.yml
$ kubectl create -f ./deployments/mongodb-deployment.yml
$ kubectl create -f ./deployments/mongodb-service.yml
```

Give these a few moments, but you can check the status of your mongodb containers, by running:

```bash
$ kubectl get pods
```

You might notice one of your pods is 'pending'. If you run `$ kubectl describe node` you might see an error regarding insufficient CPU. Unforatunately some of the cluster management and Kubernetes tooling can be quite CPU intensive on its own. So one node might not be enough for that as well as a clustered mongo instance.

So we're going to turn auto-scaling on for our cluster, which defaults to one pool. In order to do this, head over to your Google Cloud Console, select Kubernetes engine, edit your instance, turn on auto-scaling and set the min to one and the max to two, and hit save.

![Screen-Shot-2018-03-17-at-20.36.17](/content/images/2018/03/Screen-Shot-2018-03-17-at-20.36.17.png)


After a few minutes, your nodes will scale to two and you'll see 'ContainerCreating' when you run `$ kubectl get pods`. Until all of your containers are running as expected.

Now that we have our database cluster, and an auto-scaling Kubernetes engine, let's start deploying some services!


## Vessel service

The vessel service is very light-weight, it doesn't do too much, or depend on anything else, so that seems like a good place to start.

First of all, we need to make a few small code changes to our vessel-service code.

```go
// shippy-vessel-service/main.go
import (

    ...
    k8s "github.com/micro/kubernetes/go/micro"   
)

func main() {
    ...
    // Replace existing service var with...
    srv := k8s.NewService(
		micro.Name("shippy.vessel"),
		micro.Version("latest"),
	)
}
```

All we've done here, is replace the existing `micro.NewService()` with this new library we imported, `k8s.NewService()`. So what's this new library?

## Micro on Kubernetes
One of the things I love about micro, is that it was built with a great understanding of the cloud and has adapted along the way to new technologies. Micro have really taken Kubernetes seriously, and so, have created a [Kubernetes library](https://github.com/micro/kubernetes/) for micro.

Under the hood, all this library really is, is micro, configured with a sensible set of defaults for Kubernetes, and a service selector which integrates directly on-top of Kubernetes services. Which means it offloads service discovery entirely to Kubernetes. It also default to use gRPC as the default transport. Of course, you can override these behaviours using environment variables and plugins, too.

I can also say that there's some really exicting work in this space coming up from Micro, that I'm super excited about. Make sure you get involved on the [slack channel](http://slack.micro.mu/)!

Now, let's create a deployment file for our service, we'll go into a little more detail this time about what each section does.

```yaml
// shippy-vessel-service/deployments/deployment.yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: vessel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vessel
  template:
    metadata:
      labels:
        app: vessel
    spec:
        containers:
        - name: vessel-service
          image: eu.gcr.io/<project-name>/vessel-service:latest
          imagePullPolicy: Always
          command: [
            "./shippy-vessel-service",
            "--selector=static",
            "--server_address=:8080",
          ]
          env:
          - name: DB_HOST
            value: "mongo:27017"
          - name: UPDATED_AT
            value: "Mon 19 Mar 2018 12:05:58 GMT"
          ports:
          - containerPort: 8080
            name: vessel-port
```

There's a lot going on here, but I'll try and break some of it down a little more. Firs you'll see `kind: Deployment`, there are many different 'things' in Kubernetes, of which can almost be viewed as 'cloud primatives'. In programming languages, you have strings, integers, structs, methods, etc. These are primatives. Well Kubernetes views the cloud in the same way. So view these as your primatives. In this case, a deployment, which is a form of controller primative. Controllers ensure that your desired state is maintained correctly. A deployment is a form of stateless controller, they are ephemeral, things aren't lost of disrupted as they restart, or exit. Stateful sets are like deployments, except they maintain some static data and state as explained earlier. But our services shouldn't contain any state, microservices should be stateless. Thus we're using a deployment here.

Next you have a spec block, this starts with some metadata about the deployment, its name, how many of these pods (replicas), should it maintain (if one of these die, assuming you're using more than one, it's the controllers job to check the desired number of pods are running, and starts another to maintain the desired state if not). Selectors and templates expose some metadata about this pod, which allow it to be found and connected to by services.

Then you have another spec block (this is kinda confusing, but hey!). This time for our individual containers, or volumes, shared metadata etc. In this service, we're just spinning up a single container. The containers field is an array, this is because we can start several containers as part of a pod. The point is to group related containers.

The container metadata here is pretty self-explanatory, we're starting a docker container, from an image, we're setting some environment variables, passing in some commands at run time, and exposing a port (which our service will look for).

You'll notice I passed in a new command: `--selector=static`, this tells the Kubernetes micro set-up to use Kubernetes for its service discovery and load-balancing. This is really cool, because now your micro code is interacting directly with Kubernete's powerful DNS, networking, load-balancing and service discovery.

You can ommit this option and continue to use micro as previous. But we might as well get the benefits of Kubernetes here.

You'll also notice we're pulling our docker image from a private repository. When you use Google's container tools, you get a private container registry, which you can use by building your docker image, and pushing it, like so...

```bash
$ docker build -t eu.gcr.io/<your-project-name>/vessel-service:latest .
$ gcloud docker -- push eu.gcr.io/<your-project-name>/vessel-service:latest
```

Now for our service...

```yaml
// shippy-vessel-service/deployments/service.yml
apiVersion: v1
kind: Service
metadata:
  name: vessel
  labels:
    app: vessel
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    app: vessel
```

Here, as mentioned above we have a `kind`, which in this case is a service primative (a group of network level DNS and firewall rules essentially). Then we give the service a name and a label. The spec allows us to define a port for the service, and you can also define a `targetPort` here to look for a specific container. But we're doing this automatically thanks to the Kubernetes/micro implementation. Then finally, the selector, this is very important, this must match the name of the pod you want it to target, otherwise the service won't be able to find anything to proxy, and it wont work.

Now let's deploy these changes to our cluster.

```bash
$ kubectl create -f ./deployments/deployment.yml
$ kubectl create -f ./deployments/service.yml
```

Give it a few seconds, then run...

```bash
$ kubectl get pods
$ kubectl get services
```

And you should be able to see your new pod, and your new service. Ensure these are running as expected.

If you do see an error, you can run `$ kubectl proxy`, then open `http://localhost:8001/ui` in your browser, to see the Kubernetes ui, this will give you the ability to explore, in-depth, the state of your containers etc.

One thing worth mentioning here, is that deployments are atomistic and immutable, meaning they have to be changed in some way to be updated. They are turned into a unique hash, and if that hash isn't changed, then the deployment won't be updated.

If you run `$ kubectl replace -f ./deployments/deployment.yml`, nothing will happen. This is because Kubernetes has detected that nothing's changed.

There are many ways of getting around this, but it's worth noting that for the most part, it will be your container that will have changed, so instead of using the label `latest`, you should give each container a unique label, such as a build number, for example: `vessel-service:<build-no>`. This will be picked up as a change and the deployment will be replaced.

But in this tutorial, we're going to do something a little naughty, but be wary that this is lazy, and not particularly best practice. I've created a new file `deployments/deployment.tmpl`, which will act as the base deployment template. Then I've set an environment variable called `UPDATED_AT`, with a value of `{{ UPDATED_AT }}`. I've updated the Makefile to open this template file, sed the current date/time over the value of that environment variable, and output it to a final deployment.yml file. This is a bit of a hack, but it will do for now. I've seen various ways of doing this, do whatever you feel's appropriate for you.

```make
// shippy-vessel-service/Makefile
deploy:
	sed "s/{{ UPDATED_AT }}/$(shell date)/g" ./deployments/deployment.tmpl > ./deployments/deployment.yml
	kubectl replace -f ./deployments/deployment.yml
```

So there we have it, that's one service deployed and running as expected!

I'm now going to do the same for our other services. I'll update the repo's for each of those for brevity, please take a look here...

[Consignment service](https://github.com/EwanValentine/shippy-consignment-service)
[Email service](https://github.com/EwanValentine/shippy-email-service)
[User service](https://github.com/EwanValentine/shippy-user-service)
[Vessel service](https://github.com/EwanValentine/shippy-vessel-service)
[UI](https://github.com/EwanValentine/shippy-ui)

Postgres deployment for our user service...

```yaml
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  selector:
    matchLabels:
      app: postgres
  replicas: 3
  template:
    metadata:
      labels:
        app: postgres
        role: postgres
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: postgres
          image: postgres
          ports:
            - name: postgres
              containerPort: 5432
          volumeMounts:
            - name: postgres-persistent-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
         storage: 10Gi
```

Postgres service...

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
  clusterIP: None
  selector:
    role: postgres
```

Postgres storage...
```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```


## Deploying micro

```yaml
// shippy-infrastructure/deployments/micro-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: micro
spec:
  replicas: 3
  selector:
    matchLabels:
      app: micro
  template:
    metadata:
      labels:
        app: micro
    spec:
        containers:
        - name: micro
          image: microhq/micro:kubernetes
          args:
            - "api"
            - "--handler=rpc"
            - "--namespace=shippy"
          env:
          - name: MICRO_API_ADDRESS
            value: ":80"
          ports:
          - containerPort: 80
            name: port
```

And now a service...

```yaml
// shippy-infrastructure/deployments/micro-service.yml
apiVersion: v1
kind: Service
metadata:
  name: micro
spec:
  type: LoadBalancer
  ports:
  - name: api-http
    port: 80
    targetPort: "port"
    protocol: TCP
  selector:
    app: micro
```

In our service here, we're using a `LoadBalancer` type, this exposes an external load balancer, with an IP address out to the public. If you run `$ kubectl get services`, after a minute or two (you'll see 'pending' for a while), you'll get an ip address. This is public, and you can assign to a domain name etc.

Once all that's deployed, make a service call to micro:

```bash
$ curl localhost/rpc -XPOST -d '{
    "request": {
        "name": "test",
        "capacity": 200,
        "max_weight": 100000,
        "available": true
    },
    "method": "VesselService.Create",
    "service": "vessel"
}' -H 'Content-Type: application/json'
```

You should see a response, with 'created: true'. Pretty neat! That's your gRPC services, being proxied and converted to a web friendly format, using a sharded, mongodb instance. Not a huge amount of effort!

## Deploying the UI

That's our services deployed and happy, let's deploy our user interface.

```yaml
// shippy-ui/deployments/deployment.yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
        containers:
        - name: ui-service
          image: ewanvalentine/ui:latest
          imagePullPolicy: Always
          env:
          - name: UPDATED_AT
            value: "Tue 20 Mar 2018 08:26:39 GMT"
          ports:
          - containerPort: 80
            name: ui

```

Now our service...
```yaml
// shippy-ui/deployments/service.yml
apiVersion: v1
kind: Service
metadata:
  name: ui
  labels:
    app: ui
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 'ui'
  selector:
    app: ui
```

You'll notice this service is a load balancer on port 80, that's because this is a public user interface, this is how our users interact with our services. Fairly self-explanatory!


## Wrapping up

So there we have it, we've successfully ported and deployed our entire stack to the cloud using docker containers and Kubernetes to manage our containers. Hopefully you can see the value in this, and didn't find it too overwhelming.

Next part in the series, we'll look at hooking all of this up into a CI process in order to mangage our deployments.
