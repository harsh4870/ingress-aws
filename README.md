# ingress-aws
ingress-aws
Consider an AWS setup with one EC2 instance backing a public-facing Elastic Load Balancer (ELB). The gateway for the traffic in this case would be the ELB. The gateway manager is the entity that configures the ELB and runs it.


Nginx is a great choice of reverse proxy for Kubernetes
Note: I’ve used the Short format to represent Kubernetes resources. It is easier on the eyes. You can find more information about it here.

Load Balancer Planning
There are many ways to provision a load balancer for a cluster. Kubernetes lets you automate and program it to fit your needs precisely. In general, one of the following two models are used:

Setup a cluster-level load balancer and use SNI or path-based routing to services
Setup a load balancer for each service that needs it
The first approach is the one followed in this blog post. I find that most teams maintaining the load balancer are not the same as the ones managing applications. The first approach works better for such scenarios.

Management on AWS Kubernetes clusters
In this blog post, we’ll use the official nginx ingress controller residing at this repository. This is a highly versatile and configurable version of the nginx ingress controller. It can be configured in any way that nginx itself can be configured.

This ingress controller works by first creating a ELB instance for the cluster. This property allows us to easily connect the nginx ingress controller to route53.

Let’s understand this by setting up a fully functional nginx ingress controller and ingress instances.

Setup of NGINX ingress controller
Start by creating a namespace for the ingress controller to run in

kubectl create -f namespace.yaml
