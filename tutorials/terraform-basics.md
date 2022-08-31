# Terraform Basics
## Terraform motivation

When deploying software solutions, there are a few top-level components. There's (1) the hardware and (2) the software. This tutorial is all about (1) the "hardware".

These days with cloud infrastructure, there isn't really such a thing as setting up physical machines and networks. However, you do pretty much the same thing (logically) by provisioning virtual infrastructure on AWS, GCP, or Azure.

In order to do this, there are two main ways:

1. Click buttons in the console to create machines, buckets, networks, etc.
2. Declare the desired state of your infrastructure using configurations

Any time we deploy something to production, we choose (2). Never, ever, ever will we choose (1). The benefits of this are manyfold and I'd invite you to read everything possible about the benefits of "infrastructure as code".

Our preferred way to provision infrastructure is using Terraforms. There are other options such as CloudFormation but they are all more difficult to use and less robust than Terraform.

## Terraform basics

The docs for terraform are terrific so we will choose to use the material provided and give you a reading list. Then we will give you exercises that you actually perform yourself.

Note that we expect you to have a personal AWS account for the exercises. You are an independent contractor after all ;-)

1. [Terraform introduction](https://www.terraform.io/intro/index.html)
    - Don't worry about using anything SaaS provided by terraform. Use AWS S3 for remote states.
1. Go through the entire [introduction guide](https://learn.hashicorp.com/terraform/getting-started/intro). Be sure to actually do everything that they recommend!

Pay special attention to variables and remote states with going through the introduction guide.

## Exercises

### Exercise 1

If you don't have AWS VPC experience, you're gonna wanna take some hours to go check out [this tutorial](https://romandc.com/zappa-django-guide/aws_network_primer/).

Stand up the following infrastructure with terraforms:

1. A VPC
    1. Ports 80, 443, and 22 are open to the world. All else are closed.
    1. The ec2 machine has access to the postgres RDS. The RDS is not publicly available.
1. Inside the VPC is an EC2 machine (smallest one possible)
1. Also inside the VPC is an RDS postgres machine (smallest one possible)

### Exercise 2

1. Add names to all of your resources AFTER it's been deployed. This means you'll have to `apply` them.