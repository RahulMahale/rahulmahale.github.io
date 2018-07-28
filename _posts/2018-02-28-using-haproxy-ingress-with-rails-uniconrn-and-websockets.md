---
layout: post
title: Deploying Ruby on Rails application using HAProxy Ingress with unicorn/puma and websockets
published: true
categories: [Kubernetes]
description: Deploying Ruby on Rails application using HAProxy Ingress with unicorn/puma and websockets on Kubernetes.
author_github: rahulmahale
---

After months of testing we recently moved
a Ruby on Rails application to production that is using Kubernetes cluster.

In this article we will discuss
how to setup path based routing for
a Ruby on Rails application
in kubernetes using HAProxy ingress.

This post assumes that you have basic understanding of
[Kubernetes](http://kubernetes.io/)
terms like
[pods](http://kubernetes.io/docs/user-guide/pods/),
[deployments](http://kubernetes.io/docs/user-guide/deployments/),
[services](https://kubernetes.io/docs/concepts/services-networking/service/),
[configmap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
and
[ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).


Typically our Rails app has services like unicorn/puma,
sidekiq/delayed-job/resque, Websockets and some dedicated API services.
We had one web service exposed to the world
using load balancer and it was working well.
But as the traffic increased it became necessary
to route traffic based on URLs/path.

However Kubernetes does not supports
this type of load balancing out of the box.
There is work in progress for [alb-ingress-controller](https://github.com/coreos/alb-ingress-controller)
to support this
but we could not rely on it for
production usage as it is still in alpha.

The best way to achieve path based routing was to use
[ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers).

We researched and found that there are different types of ingress
available in k8s world.

 1. [nginx-ingress](https://github.com/kubernetes/ingress-nginx)
 2. [ingress-gce](https://github.com/kubernetes/ingress-gce)
 3. [HAProxy-ingress](https://github.com/jcmoraisjr/haproxy-ingress)
 4. [traefik](https://docs.traefik.io/configuration/backends/kubernetes/)
 5. [voyager](https://github.com/appscode/voyager)

We experimented with nginx-ingress and HAProxy and decided to go with HAProxy.
HAProxy has better support for Rails websockets which we needed in the project.

We will walk you through step by step on how to use haproxy ingress in a Rails app.

### Configuring Rails app with HAProxy ingress controller

Here is what we are going to do.

* Create a Rails app with different services and deployments.
* Create tls secret for SSL.
* Create HAProxy ingress configmap.
* Create HAProxy ingress controller.
* Expose ingress with service type LoadBalancer
* Setup app DNS with ingress service.
* Create different ingress rules specifying path based routing.
* Test the path based routing.

Now let's build Rails application deployment manifest
for services like web(unicorn),background(sidekiq),
Websocket(ruby thin),API(dedicated unicorn).

Here is our web app deployment and service template.

{% highlight yaml %}

---
apiVersion: v1
kind: Deployment
metadata:
  name: test-production-web
  labels:
    app: test-production-web
  namespace: test
spec:
  template:
    metadata:
      labels:
        app: test-production-web
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-production
        imagePullPolicy: Always
       env:
        - name: POSTGRES_HOST
          value: test-production-postgres
        - name: REDIS_HOST
          value: test-production-redis
        - name: APP_ENV
          value: production
        - name: APP_TYPE
          value: web
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: registrykey
---
apiVersion: v1
kind: Service
metadata:
  name: test-production-web
  labels:
    app: test-production-web
  namespace: test
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test-production-web

{% endhighlight %}

Here is background app deployment and service template.

{% highlight yaml %}

---
apiVersion: v1
kind: Deployment
metadata:
  name: test-production-background
  labels:
    app: test-production-background
  namespace: test
spec:
  template:
    metadata:
      labels:
        app: test-production-background
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-production
        imagePullPolicy: Always
       env:
        - name: POSTGRES_HOST
          value: test-production-postgres
        - name: REDIS_HOST
          value: test-production-redis
        - name: APP_ENV
          value: production
        - name: APP_TYPE
          value: background
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: registrykey
---
apiVersion: v1
kind: Service
metadata:
  name: test-production-background
  labels:
    app: test-production-background
  namespace: test
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test-production-background

{% endhighlight %}

Here is websocket app deployment and service template.

{% highlight yaml %}

---
apiVersion: v1
kind: Deployment
metadata:
  name: test-production-websocket
  labels:
    app: test-production-websocket
  namespace: test
spec:
  template:
    metadata:
      labels:
        app: test-production-websocket
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-production
        imagePullPolicy: Always
       env:
        - name: POSTGRES_HOST
          value: test-production-postgres
        - name: REDIS_HOST
          value: test-production-redis
        - name: APP_ENV
          value: production
        - name: APP_TYPE
          value: websocket
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: registrykey
---
apiVersion: v1
kind: Service
metadata:
  name: test-production-websocket
  labels:
    app: test-production-websocket
  namespace: test
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test-production-websocket

{% endhighlight %}

Here is API app deployment and service info.

{% highlight yaml %}

---
apiVersion: v1
kind: Deployment
metadata:
  name: test-production-api
  labels:
    app: test-production-api
  namespace: test
spec:
  template:
    metadata:
      labels:
        app: test-production-api
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-production
        imagePullPolicy: Always
       env:
        - name: POSTGRES_HOST
          value: test-production-postgres
        - name: REDIS_HOST
          value: test-production-redis
        - name: APP_ENV
          value: production
        - name: APP_TYPE
          value: api
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: registrykey
---
apiVersion: v1
kind: Service
metadata:
  name: test-production-api
  labels:
    app: test-production-api
  namespace: test
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test-production-api

{% endhighlight %}

Let's launch this manifest using `kubectl apply`.

{% highlight bash %}

$ kubectl apply -f test-web.yml -f test-background.yml -f test-websocket.yml -f test-api.yml
deployment "test-production-web" created
service "test-production-web" created
deployment "test-production-background" created
service "test-production-background" created
deployment "test-production-websocket" created
service "test-production-websocket" created
deployment "test-production-api" created
service "test-production-api" created

{% endhighlight %}

Once our app is deployed and running we should create HAProxy ingress.
Before that let's create a tls secret with our SSL key and certificate.

This is also used to enable HTTPS for app URL and to terminate it on L7.

{% highlight bash %}

$ kubectl create secret tls tls-certificate --key server.key --cert server.pem

{% endhighlight %}

Here `server.key` is our SSL key and
`server.pem` is our SSL certificate in pem format.

Now let's Create HAProxy controller resources.

### HAProxy configmap

For all the available configuration parameters from HAProxy refer [here](https://github.com/jcmoraisjr/HAProxy-ingress#configmap).

{% highlight yaml %}

apiVersion: v1
data:
    dynamic-scaling: "true"
    backend-server-slots-increment: "4"
kind: ConfigMap
metadata:
  name: haproxy-configmap
  namespace: test

{% endhighlight %}


### HAProxy Ingress controller deployment

Deployment template for the Ingress controller with at-least
2 replicas to manage rolling deploys.

{% highlight yaml %}

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: haproxy-ingress
  name: haproxy-ingress
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      run: haproxy-ingress
  template:
    metadata:
      labels:
        run: haproxy-ingress
    spec:
      containers:
      - name: haproxy-ingress
        image: quay.io/jcmoraisjr/haproxy-ingress:v0.5-beta.1
        args:
        - --default-backend-service=$(POD_NAMESPACE)/test-production-web
        - --default-ssl-certificate=$(POD_NAMESPACE)/tls-certificate
        - --configmap=$(POD_NAMESPACE)/haproxy-configmap
        - --ingress-class=haproxy
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: stat
          containerPort: 1936
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

{% endhighlight %}

Notable fields in above manifest are arguments passed to controller.

` --default-backend-service` is the service
when No rule is matched your request will be served by this app.

In our case it is `test-production-web` service,
But it can be custom 404 page or whatever better you think.

`--default-ssl-certificate` is the SSL secret we just
created above this will terminate SSL on L7 and our app
is served on HTTPS to outside world.

### HAProxy Ingress service

This is the `LoadBalancer` type service to allow
client traffic to reach our Ingress Controller.

LoadBalancer has access to both public network and
internal Kubernetes network while retaining
the L7 routing of the Ingress Controller.

{% highlight yaml %}

apiVersion: v1
kind: Service
metadata:
  labels:
    run: haproxy-ingress
  name: haproxy-ingress
  namespace: test
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  - name: stat
    port: 1936
    protocol: TCP
    targetPort: 1936
  selector:
    run: haproxy-ingress

{% endhighlight %}


Now let's apply all the manifests of HAProxy.


{% highlight bash %}

$ kubectl apply -f haproxy-configmap.yml -f haproxy-deployment.yml -f haproxy-service.yml
configmap "haproxy-configmap" created
deployment "haproxy-ingress" created
service "haproxy-ingress" created

{% endhighlight %}

Once all the resources are running get the LoadBalancer endpoint using.

{% highlight bash %}

$ kubectl -n test get svc haproxy-ingress -o wide

NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                            PORT(S)                                     AGE       SELECTOR
haproxy-ingress   LoadBalancer   100.67.194.186   a694abcdefghi11e8bc3b0af2eb5c5d8-806901662.us-east-1.elb.amazonaws.com   80:31788/TCP,443:32274/TCP,1936:32157/TCP   2m        run=ingress

{% endhighlight %}

### DNS mapping with application URL

Once we have ELB endpoint of ingress service,
map the DNS with URL like `test-rails-app.com`.

### Ingress Implementation

Now after doing all the hard work it is time to configure
ingress and path based rules.

In our case we want to have following rules.

*https://test-rails-app.com* requests to be served by `test-production-web`.

*https://test-rails-app.com/websocket* requests to be served by `test-production-websocket`.

*https://test-rails-app.com/api* requests to be served by `test-production-api`.

Let's create a ingress manifest defining all the rules.


{% highlight yaml %}
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  namespace: test
spec:
  tls:
    - hosts:
      - test-rails-app.com
      secretName: tls-certificate
  rules:
    - host: test-rails-app.com
      http:
        paths:
          - path: /
            backend:
              serviceName: test-production-web
              servicePort: 80
          - path: /api
            backend:
              serviceName: test-production-api
              servicePort: 80
          - path: /websocket
            backend:
              serviceName: test-production-websocket
              servicePort: 80

{% endhighlight %}

Moreover there are [Ingress Annotations](https://github.com/jcmoraisjr/haproxy-ingress#annotations)
for adjusting configuration changes.

As expected,now our default traffic on
`/` is routed to `test-production-web` service.

`/api` is routed to `test-production-api` service.

`/websocket` is routed to `test-production-websocket` service.

Thus ingress implementation solves our purpose of path based routing
and terminating SSL on L7 on Kubernetes.
