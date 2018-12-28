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

STEP : 1
kubectl create -f namespace.yaml

Step:2
Then create the cluster role and role for the nginx ingress controller

kubectl create -f rbac.yaml

This will create the cluster role and role for the ingress controller. Then, write a configuration file for configuring your nginx proxy

config_map:
  data:
    client-body-buffer-size: 32M
    hsts: "true"
    proxy-body-size: 1G
    proxy-buffering: "false"
    proxy-read-timeout: "600"
    proxy-send-timeout: "600"
    server-tokens: "false"
    ssl-redirect: "false"
    upstream-keepalive-connections: "50"
    use-proxy-protocol: "true"
  labels:
    app: ingress-nginx
  name: nginx-configuration
  namespace: ingress-nginx
  version: v1
---
config_map:
  name: tcp-services
  namespace: ingress-nginx
  version: v1
---
config_map:
  name: udp-services
  namespace: ingress-nginx
  version: v1
Note the configuration set above. This is a standard config that has worked well for us.

It is recommended to use HTTP Strict Transport Security (hsts). This, along with SSL redirects, are used to redirect HTTP requests to HTTPS. This is generally considered to be more secure. We recommend it unless your service specifically needs to terminate SSL itself.

The other important option above is use-proxy-protocol. Since I’ll be setting up a L4 ELB loadbalancer (which does not forward SRC IP, SRC Port, SRC proto and other possibly important information to the services behind it) this option provides a mechanism to forward those L7 headers.

Let’s create this config map

$ short -k -f configmap.short.yaml > configmap.yaml
$ kubectl create -f configmap.yaml
Now that the configuration is ready to be used, we can start by creating the default backend. The default backend acts as a catch-all service. It is routed to whenever an unknown URL is requested from this proxy (i.e. the nginx proxy we’ll be running).

We’re almost done. The proxy is present and running. However, to route traffic from outside the cluster using a single URL, we need to create a ELB instance. This can be done by creating a Kubernetes Service of type Loadbalancer.

apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: '*'
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:1234567:certificate/1234294-232-4f89-bca8
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: https
  labels:
    k8s-addon: ingress-nginx.addons.k8s.io
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: http
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app: ingress-nginx
  type: LoadBalancer
Note that I’ve set a ssl cert (emphasized above). This is the cert for terminating SSL and should belong to the domain that you own. We shall see how to route traffic from this domain to specific services later in this article. For the sake of this example, let’s consider this domain to be:

kube-example.com

AWS certificate manager can be used to create a cert for this domain. Once created, note the arn for the cert and use that here to terminate SSL in the ELB. Now, Let’s deploy this Loadbalancer service

$ short -k -f nginx-service.short.yaml > nginx-service.yaml
$ kubectl create -f nginx-service.yaml
The cluster will have a fully functional nginx load balancer fronted by ELB.

Setting up Route 53 and Redirecting traffic
Note that this part is not required for simple testing. You can use the ELB public URLs to test the routing capabilities.

Now that the ELB is up and running, you can configure a domain/sub-domain in route53 to point to this load balancer. Instructions for this can be found here.

Once setup, you can create services and redirect traffic to specific paths using ingress resources.

Let’s understand this with an example. As a quick recap, the domain we’re working with is kube-example.com

Consider the example of exposing the Grafana dashboard for your Kubenretes cluster at the path /monitor. This can be achieved by created an ingress resource:

ingress:
  annotations:
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-allow-headers: Authorization, origin, accept
    nginx.ingress.kubernetes.io/cors-allow-methods: GET, OPTIONS
    nginx.ingress.kubernetes.io/cors-allow-origin: kube-example.com/monitor
    nginx.ingress.kubernetes.io/enable-cors: "true"
  name: kubernetes-grafana-ingress
  namespace: kube-system
  rules:
  - host: kube-example.com
    paths:
    - path: /monitor
      port: 80
      service: monitoring-grafana
  tls:
  - hosts:
    - kube-example.com
  version: extensions/v1beta1
Note the highlighted annotation. This annotation tells ingress to rewrite the path that is sent to grafana. The incoming request will be redirect as shown below.

What User Requests: kube-example.com/monitor
What Grafana gets: kube-example.com/
Also grafana requires CORS support. The configuration to add this is shown in the Ingress annotations above.

