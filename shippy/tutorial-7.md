# Microservices with Golang and Micro - Part 7

In the [previous post](https://ewanvalentine.io/microservices-in-golang-part-6/), we touched briefly on user interfaces and web clients and how to interact with our newly created rpc services using the micro toolkits rpc proxy.

This post, we'll look at how to create a cloud environment to host our services. We'll be using [Terraform](https://www.terraform.io/) to architect our cloud cluster on Google Cloud platform. This should be a fairly short post, but it's an important one.


## Why Terraform
I've used a few different cloud provisioning solutions, but Hashicorps [Terraform](https://www.terraform.io/), to me feels the easiest to use and the best supported. There's a term that's cropped up in recent years: '[infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_Code)'. Why would you want your infrastructure as code? Well, infrastructure is complex, it describes a lot of moving parts. It's also important to track changes and version control changes to your infrastructure.

[Terraform](https://www.terraform.io/) does this beautifully. They have actually created their own DSL (domain specific language), which allows you to describe what your infrastructure should look like.

[Terraform](https://www.terraform.io/) also allows you to make atomic changes. So in the case of an error of failure, it will back everything off and roll it back to how it was. Terraform even allows you to preview changes by doing a roll-out plan. Which will describe exactly what your changes will do to your infrastructure. This gives you a lot of confidence in what used to be a scary prospect.

So let's get started!


## Creating your cloud project

Head over to [Google Cloud](http://console.cloud.google.com/) and create a new project. If you haven't used it before, you'll probably find you have a £300 free trial. Nice! Anyway, you should see a blank new project. Now on your left, you should see an IAM & Admin tabb, head there and create a new service key. Ensure 'furnish new key' is selected, then make sure you select the json type. Keep this safe, we'll need this later. This allows programs to execute changes to the Google Cloud API's on your behalf. Now we should have just about everything we need to get started.

![Screen-Shot-2018-02-10-at-10.58.07](/content/images/2018/02/Screen-Shot-2018-02-10-at-10.58.07.png)

So create yourself a new repo. I've called mine shippy-infrastructure.


## Describing our infrastructure

Create a new file called `variables.tf`. This will house the basic information about our project. Here we have our project id, our region, our project name and our platform name. The region is datacenter we want our cluster to run on. The zone is the availability zone within that region. The project name is the project id for our Google project, and finally, the platform name is the name of our cluster.

```clike
variable "gcloud-region"    { default = "europe-west2" }
variable "gcloud-zone"      { default = "europe-west2-a" }
variable "gcloud-project"   { default = "shippy-freight" }
variable "platform-name"    { default = "shippy-platform" }
```

Create a new file called `providers.tf`, this does the Google specific part:

```clike
provider "google" {
  credentials = "${file("google-cred.json")}"
  project     = "${var.gcloud-project}"
  region      = "${var.gcloud-region}"
}
```

Now let's create a file called `global.tf`. Here's the chunky part of our set-up:

```clike
# Creates a network layer
resource "google_compute_network" "shippy-network" {
  name = "${var.platform-name}"
}

# Creates a firewall with some sane defaults, allowing ports 22, 80 and 443 to be open
# This is ssh, http and https.
resource "google_compute_firewall" "ssh" {
  name    = "${var.platform-name}-ssh"
  network = "${google_compute_network.shippy-network.name}"

  allow {
    protocol = "icmp"
  }

  allow {
    protocol = "tcp"
    ports    = ["22", "80", "443"]
  }

   source_ranges = ["0.0.0.0/0"]
}

# Creates a new DNS zone
resource "google_dns_managed_zone" "shippy-freight" {
  name        = "shippyfreight-com"
  dns_name    = "shippyfreight.com."
  description = "shippyfreight.com DNS zone"
}

# Creates a new subnet for our platform within our selected region
resource "google_compute_subnetwork" "shippy-freight" {
  name          = "dev-${var.platform-name}-${var.gcloud-region}"
  ip_cidr_range = "10.1.2.0/24"
  network       = "${google_compute_network.shippy-network.self_link}"
  region        = "${var.gcloud-region}"
}

# Creates a container cluster called 'shippy-freight-cluster'
# Attaches new cluster to our network and our subnet,
# Ensures at least one instance is running
resource "google_container_cluster" "shippy-freight-cluster" {
  name = "shippy-freight-cluster"
  network = "${google_compute_network.shippy-network.name}"
  subnetwork = "${google_compute_subnetwork.shippy-freight.name}"
  zone = "${var.gcloud-zone}"

  initial_node_count = 1

  master_auth {
    username = <redacted>
    password = <redacted>
  }

  node_config {

    # Defines the type/size instance to use
    # Standard is a sensible starting point
    machine_type = "n1-standard-1"

    # Grants OAuth access to the following API's within the cluster
    oauth_scopes = [
      "https://www.googleapis.com/auth/projecthosting",
      "https://www.googleapis.com/auth/devstorage.full_control",
      "https://www.googleapis.com/auth/monitoring",
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/compute",
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}

# Creates a new DNS range for cluster
resource "google_dns_record_set" "dev-k8s-endpoint-shippy-freight" {
  name  = "k8s.dev.${google_dns_managed_zone.shippy-freight.dns_name}"
  type  = "A"
  ttl   = 300

  managed_zone = "${google_dns_managed_zone.shippy-freight.name}"

  rrdatas = ["${google_container_cluster.shippy-freight-cluster.endpoint}"]
}
```

Now move the key you created earlier into the root of this project and name it `google-cred.json`.

You should now have everything you need in order to create a new cluster! But let's not go crazy, you should run a test first and check all is well.

Run `$ terraform init` - this will download any missing providers/plugins. This will pick up that we're using the Google Cloud module and automatically fetch those dependencies.

Now if you run `$ terraform plan` it will show you exactly what changes it will make. This is almost like doing `$ git diff` on your entire infrastructure. Now that is cool!

Having glanced over the deployment plan, I think we're good to go.

`$ terraform apply`

*Note: you may get asked to enable a few api's in order to complete this, that's fine, click the links, ensure you enable them and re-run `$ terraform apply`. If you want to save time, head over to the API's section in the Google Cloud console, and enable the DNS, Kubernetes and Compute Engine API's.*

This may take a while, but that's because a lot is happening. But once it's complete, you should be able to see your new cluster if you head over to Kubernetes engine or Compute Engine segments in the right hand menu in the Google Cloud console.

*Note: if you are not making use of a free-trial period, this will immediately start costing you money. Be sure to look at the instance pricing list. Oh, and I've powered mine down between working on these posts. This will not incur charges, as these are charged on a resource usage basis.*

![Screen-Shot-2018-02-10-at-12.25.11-1](/content/images/2018/02/Screen-Shot-2018-02-10-at-12.25.11-1.png)

That's it! We have a fully functioning cluster/vm, ready for us to start deploying our containers to. The next part in this series will be on Kubernetes, and how to set-up and deploy containers to Kubernetes.
