# Ansible basics

As you learned from the [Terraform Basics](./terraform-basics.md) tutorial, there are two distinct parts having to do with deployment. Terraform is the "hardware" portion of deployment. So after we have happily `terraform apply`d and have some network, machines, etc. set up, we'll need to install and run some software on that in order for it to actually be useful. In cases where we don't need explicitly need autoscaling immediately, ansibles are a great tool for this. They read very easily and anything you can do manually can pretty easily be put into an ansible.

Ansible has been around for a looooong time. Like a long time. It is very robust. It might feel a bit outdated in a few ways but when dealing with infrastructure you usually want to err on the side of robust over shiny and new.

The ansible docs are quite good. Reading through a few of them should be a enough to get you started:

1. [How ansible works](https://www.ansible.com/overview/how-ansible-works)
1. [Ansible basic concepts](https://docs.ansible.com/ansible/latest/network/getting_started/basic_concepts.html)

This should be enough to get you started. Go through it carefully!

## Exercises

For all of these exercises, be sure to use terraforms! Also, everything should be encrypted.

### Exercise 1

Deploy your own ghost blogging platform using terraform and ansible. You may use [this repo](https://github.com/DareData/example-encrypted-http-service) as a reference though you should avoid
copy-pasting things without understanding.

### Exercise 2

Choose an open source web app of your choice and deploy that yourself.

### Exercise 3

Write a little flask http server that does nothing special and deploy that as well. For this one you'll have to make some decisions about how to run it reliably.