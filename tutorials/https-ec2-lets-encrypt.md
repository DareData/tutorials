# HTTPS on EC2 with LetsEncrypt on AWS
## HTTPS on EC2 with LetsEncrypt

A very common need is to have an ec2 instance running that serves some kind of HTTP server with TLS encryption. These use cases can range from everything from deploying a dashboarding tool for internal use to deploying a production-ready HTTP server for a service that doesn't need auto-scaling.

Because our domain name system is essentially real estate, it is difficult to automate the entire thing end-to-end because of the need to show that you are a real person using scarce resources in a few places. Still, once the server is developed and you know how to run it, developing the infrastructure as code and deploying in a robust and reproducible way should be (at the most) a matter of an hour or two.

Feel free to refer to an [example setup repo](https://github.com/DareData/example-encrypted-http-service) for running a ghost blog. It covers a bit more than we'll cover here but is still very useful as a reference.

Note: if you have not gone through the [terraform basics](./terraform-basics.md) and [ansible basics](./ansible-basics.md) tutorials do that first or the rest is going to be rough times.

## Step 1: Get an IP address and ssh key

Go into your AWS account and allocate an elastic IP address. This is trivial and should take a few minutes max. Be sure to note the allocation ID for future use in your terraforms. Create an ssh key as well in the case that you don't already have one you want to use.

## Step 2: Point your domain name to the IP address.

Depends on your provider but this is trivial and should take a few minutes max.

## Step 3: Write the terraforms

### Step 3.1: Understand the `main.tf`

You'll want a `main.tf` that looks like the following:

```terraform
provider "aws" {
  profile    = "default"
  region     = "eu-west-1"
  version = "~>2.45"
}

terraform {
  required_version = "~>0.12"
  backend "s3" {
    region = "eu-west-1"
    bucket = "TODO"
    key    = "tfstates/"
  }
}

module "vpc" {
  source = "./modules/vpc"
  app_name = var.app_name
  stage = var.stage
}

module "http-server" {
  source = "./modules/http-server"
  app_name = var.app_name
  stage = var.stage
  ssh_key_name = var.ssh_key_name
  subnet_id = module.vpc.subnet_server_id_a
  vpc_security_group_ids = module.vpc.vpc_security_group_ids
  instance_type = var.instance_type
  elastic_ip_allocation_id = var.http_server_elastic_ip_allocation_id
}
```

The `provider "aws"` and `terraform` sections we won't cover as you should already be familiar with them.

Where the really interesting stuff is happening is in the `module "vpc"` and `module "http-server"` sections. What we want to parameterize in this `main.tf` are the following:

1. `app_name` - We will use this to tag all of our resources which will help with billing and generally keeping things organized later on down the road.
1. `stage` - We need to be able to deploy multiple versions of our infrastructure. We might need test, staging, and production for example and the `stage` will be used to help us track this.
1. `ssh_key_name` - The name of the key we created in step 1
1. `elastic_ip_allocation_id` - The allocation ID that we noted in step 1
1. `instance_type` - The type (a.k.a. size) of instance we are deploying

### Step 3.2: Write the `vpc` module

To gain a working understanding of AWS networking basics, check out [this tutorial](https://romandc.com/zappa-django-guide/aws_network_primer/). It's a bit of reading but very well explained and accurate.

Note that there are a lot of networking concepts here that we are taking for granted. In order to ground yourself in the networking side of AWS, check out [this excellent networking primer](https://romandc.com/zappa-django-guide/aws_network_primer/).

Inside a directory called `modules`, you'll want to have a few directories called `vpc` and `http-server`. Inside of the `vpc` directory you'll want to implement the contents of this section.

We always want to be deploying infrastructure into dedicated networks. This significantly reduces attack surfaces visible from the outside as well as help ensure that nobody internal is accidentally given access to anything that they shouldn't have access to.

`modules/vpc/main.tf` should look like the following:

```terraform
resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  tags = {
    Service  = var.app_name
    Environment = var.stage
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Service  = var.app_name
    Environment = var.stage
  }
}

resource "aws_security_group" "tls_ssh" {
  name = "${var.app_name}-${var.stage}-allow_tls_ssh"
  description = "Allow TLS and SSH inbound trafic to the machine"
  vpc_id = aws_vpc.main.id
  ingress {
    description = "Allow TLS Connections"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "Allow HTTP Connections" # Certbot needs http to verify the certificate
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "Allow SSH Connections"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
  }
  tags = {
    Service  = var.app_name
    Environment = var.stage
  }
}

resource "aws_route_table" "r" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Service  = var.app_name
    Environment = var.stage
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.main_a.id
  route_table_id = aws_route_table.r.id
}
```

Notes on the sections are as follows:

1. `resource "aws_vpc" "main"` - a VPC is a first class citizen on AWS. This actually creates a logical network that mirrors almost all of the functionality of a classical network. Think of this as some technicians came into your office and wired everything up but didn't install anything.
1. `aws_internet_gateway` - If you want your network to be able to communicate with the outside world, you need one of these. Never seen a network without it. The advantage of creating a new one here is that it won't use the default one which keeps things a bit more secure.
1. `resource "aws_security_group" "tls_ssh"` - This creates a security group that specifies rules about the type of traffic that is allowed into our network. For this one we allow 80 (normal HTTP), 443 (HTTPS), and 22 (ssh). For this setup you'll need all of them though you won't be accepting much traffic at all on port 80 when in production.
1. `resource "aws_route_table" "r"` - Check out [this section](https://romandc.com/zappa-django-guide/aws_network_primer/#hooking-things-up-route-tables) of the networking primer. Essentially the route tables set the rules for how traffic is allowed to flow through different subnets for security reasons.
1. `resource "aws_route_table_association" "a"` - Route tables are just lists of rules. They don't do anything until they are assigned to a subnet and that is what this stanza does.

### Step 3.3: Write the `http-server` module

In the `http-server` module directory we have a `main.tf` like this:

```terraform
data "aws_caller_identity" "current" {}

data "aws_ami" "ubuntu" {
  most_recent = true

  owners = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  private_ip = var.server_private_ipv4
  subnet_id = var.subnet_id
  security_groups = var.vpc_security_group_ids
  key_name = var.ssh_key_name
  tags = {
    Name = "${var.app_name}-${var.stage}-http-server"
    Service  = var.app_name
    Environment = var.stage
  }
}

resource "aws_eip_association" "eip_assoc" {
  instance_id   = aws_instance.web.id
  allocation_id = var.elastic_ip_allocation_id
}

```

The interesting points are:

- `data "aws_ami" "ubuntu"` - Select the most recent LTS server always! Ubuntu is not necessarily the most efficient though it is very widely used and usually ends up having more support / being easier to use.
- `resource "aws_instance" "web"` - actually create the right machine and place it into the correct VPC that was created in the previous step
- `resource "aws_eip_association" "eip_assoc"` - This assigns the IP address that you created earlier to this machine. Absolutely critical.

Not much more here, this section is usually super simple.

## Step 4: Write the ansibles

Terraforms are all about provisioning infrastructure (the "hardware"). Ansible is all about installing and configuring the software on said infrastructure.

TODO: write an intro tutorial for ansibles the same way we've got it for Terraform.

You'll want three different playbooks:

1. Initial machine setup. This just gets the basics installed and is useful for pretty much any time you want to be doing anything with ansibles.
1. Install nginx and certs - this installs nginx and takes care of all the coordination with letsencrypt to get your certificate issued. It also sets up auto-renew so you don't have to worry about the certificate expiring.
1. Install app. This is where you'll actually install the application that you're trying to deploy. In this case, we'll leave it blank but you can [see what one looks like](https://gitlab.com/DareData-open-source/encrypted-http-service/-/tree/master/ansible/roles/application) for setting up the ghost blogging platform.

### Step 4.0

Fill in the `ansible/group_vars/all` file with the following:

```
# For setting up the ssl certs
domain_name: TODO
ssl_email: TODO
```

The `domain_name` is the one that you associated with the IP address earlier when setting up the terraforms. The `ssl_email` should be your email address as it is the one that should be associated with the certificate.

### Step 4.1: Initial machine setup

```ansible
---
- name: Update
  apt:
    upgrade: dist
    cache_valid_time: 3600

- name: Only run "update_cache=yes" if the last one is more than 3600 seconds ago
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install common utilities
  apt:
    pkg:
      - htop
      - git

- name: Add ssh keys
  blockinfile:
    block: "{{ lookup('file', './ssh_keys/authorized_keys') }}"
    dest: /home/ubuntu/.ssh/authorized_keys
    backup: yes

```

Everything here should be self-explanatory. The last one "Add ssh keys" is especially important since you can store the public keys of anyone that should be given ssh access to the machine in the file `./ssh_keys/authorized_keys` and it will automatically be added to the `authorized_keys` on the server.

### Step 4.2: Nginx and LetsEncrypt

We use LetsEncrypt because it is free, secure, backed by a robust non-profit, and allows for regeneration of certificates automatically. The ops are good and incentives are aligned.

In `ansible/roles/nginx/tasks/main.yml` you'll want the following:

```ansible
---
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Install software-properties-common
  apt:
    name: software-properties-common
    state: present

- name: add certbot repository
  apt_repository:
    repo: ppa:certbot/certbot

- name: Copy nginx configuration
  template:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
  notify: reload nginx

- name: apt update cache
  apt:
    update_cache: yes

- name: Install certbot
  apt:
    name:
      - certbot
      - python3-certbot-nginx
    state: present

- name: install certbot certificates and auto-renewal
  shell: certbot --nginx -m {{ ssl_email }} --agree-tos -d {{ domain_name }} --non-interactive

- name: Ensures site dir exists
  file:
    path: /var/www/{{ domain_name }}
    state: directory

- name: Copy sample index file
  template:
    src: index.html
    dest: /var/www/{{ domain_name }}/index.html
  notify: reload nginx

- meta: flush_handlers
```

This reads pretty cleanly with important points as follows:

1. This uses LetsEncrypt packages entirely to manage the certificate issuing process. This is good, we don't want to be managing that ourselves if at all possible.
1. We did create an `nginx.conf` that looks like [this](https://gitlab.com/DareData-open-source/encrypted-http-service/-/blob/master/ansible/roles/nginx/templates/nginx.conf). This is templated to be filled in with the domain name you want associated with it with two different sections:
    1. Port 80 config - this is only for use when generating the SSL cert. Never serve anything in production through this. This port needs to be open and stay open though because this is how LetsEncrypt will re-issue certificates.
    1. Port 443 config - this just includes from a standard directory. Usually the application that you're setting up will be configured using a reverse proxy that can just be stored in the `/etc/nginx/conf.d/*.conf` directory. In this way we decouple the issuing of the TLS cert from the actual application itself.

### Step 4.3: Application ansibles

Assuming that you know the commands to set up your machine, you'll want to encode them into an ansible so all future humans can benefit from your work. In the case of getting the ghost blogging platform set up, the `ansible/roles/application/tasks/main.yml` looks like this:

```ansible
---

- name: add nodejs repo
  shell: curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash

- name: Install node.js
  become: yes
  apt:
    name:
      - nodejs
    state: present

- name: Install ghost-cli
  become: yes
  npm:
    global: yes
    name: ghost-cli
    state: present

- name: Create ghost directory
  file:
    path: "{{ application_dir }}"
    state: directory
    owner: "{{ application_user }}"
    group: "{{ application_user }}"
    mode: 0755

- name: installing ghostjs
  shell: >
    ghost install
    --url "https://{{ domain_name }}"
    --port 2368
    --db mysql
    --dbhost "{{ database_host }}"
    --dbuser "{{ database_user }}"
    --dbpass "{{ database_password }}"
    --dbname "{{ database_name }}"
    --systemd
    --enable
    --start
    --no-setup-nginx
    --no-setup-ssl
    --no-prompt
  args:
    chdir: "{{ application_dir }}"

- name: Copy nginx app server block
  become: yes
  template:
    src: app.conf
    dest: /etc/nginx/conf.d/app.conf
  notify: reload nginx
```

This reads pretty well but take care to notice the end in which we copy an nginx config block into the `/etc/nginx/conf.d/` directory mentioned earlier. This block is a very simple one and looks like this:

```
location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header HOST $http_host;
    proxy_set_header X-NginX-Proxy true;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_pass http://127.0.0.1:2368;
    proxy_redirect off;
}
```

And then it requests that nginx reload in order to pick up the new configs. Setting up a reverse proxy that doesn't need auto-scaling like this is an incredibly common task and many apps will have recommended nginx proxy configs that you should look up.

# Workflow

Now that you've seen each of the individual components, let's take a minute to see what the development workflow might look like:

1. Write and run the terraforms until you get the right topology. You might need to destroy / recreate several times and that's okay. The most important thing is that you are able to stand up some new infrastructure in one click.
1. Run the machine setup and nginx ansibles to get the machine set up.
1. Manually install your application and take careful note of all the commands and steps that you took to do it.
1. Transcribe your steps into ansibles and test. During this phase, you might need to "taint" and re-deploy the ec2 machine several times in order to avoid having to re-provision ALL of the infrastructure.
1. Tear the whole thing down and re-run. Remember, this is what you'll be passing off to someone else so it needs to work as advertised.

## Exercises

Note that for both of these exercises you'll need an rds instance running.

1. Deploy a [digdag](https://www.digdag.io/) server using the above information
1. Deploy a [metabase](https://www.metabase.com/) server using the above information