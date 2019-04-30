---
title: "Securing Kubernetes Services with TLS Ingresses"
date: 2019-04-30T12:43:44-05:00
draft: false
author: "Joseph D. Marhee"
---

One of the common resources in Kubernetes for exposing ports, aiding in named-based service discovery, and creating load balancer objects in Kubernetes is the `Service` resource, which can be defined using a manifest like:

```
kind: Service
apiVersion: v1
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```

to create a service, `web-service.namespace`, resolvable within the cluster, and if you add:

```
  type: LoadBalancer
```

if exposes `<External IP>:<Port>` to the network outside the cluster. However, this is a bare IP address, and unless the port exposed is a TLS-equipped endpoint (i.e. if it terminates in the application container itself, with an Nginx frontend `Pod`, or at the LoadBalancer upstream, etc.), this will be an insecure connection for your application.

What will an `Ingress` do for me that a `Service` won't?
---

One solution for this, rather than expose via `type: LoadBalancer` in your `Service` definition, is to use an `Ingress`. There are many `Ingress` classes and providers for Kubernetes ([Traefik](traefik.io) is a personal favorite of mine, but won't cover its usage here, as it is not standard in Kubernetes), and they have different capabilities and features. The `Ingress`, like many webservers (Nginx, Apache, etc.), uses a name-based routing rule, so an Ingress can target a `Service`, like so:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: web-service
          servicePort: 80
```

where an `Ingress` like the above, to `http://{hostname}/`, would route to your `web-service` we defined earlier. You can modify the `path` key to use different URIS to direct to different backends (i.e. if you have an application architecture made of different microservices to provide different functions in an API), and it does this relative to a hostname, which you can implement using the `- host` rule:

```
...
spec:
  rules:
  - host: my-web-app.com
    http:
      paths:
      - path: /
...
```

and then create new Ingress rules for multiple hostnames (i.e. subdomains, or entirely different domains on the same cluster, provided they use the same Namespace). 

You might recognize this behavior if you've configured an Nginx `server` block, or Apache `VirtualHost` before; this is a very similar principle and similar resulting behavior, as your users see it. Nginx is, for example, one of the standard `Ingress` classes available to you, and the one we will use to implement TLS for our `Ingress` object.

Creating a Secret for your Certificate and Key
---

There are more involved, or sophisticated, methods of managing certificates in Kubernetes, but because the emphasis of this process is on the `Ingress` resource and not certificate management (which I'll cover in a later post!), I'll assume you have, either, a self-signed certificate pair or a CA-issued certificate and key pair, which we'll register with Kubernetes in a `Secret`.

You'll want to `base64`-encode (keep in mind, this data is _not_ encrypted, just encoded, so handle this data carefully--I'll cover this briefly at the end), the key and certificate:

```
cat cert.crt | base64
cat cert.key | base64
```

and place the resulting values in your `Secret` Definition:

```
apiVersion: v1
data:
  tls.crt: CERT BASE64 STRING GOES HERE
  tls.key: KEY BASE64 STRING GOES HERE
kind: Secret
metadata:
  name: app-router-tls
  namespace: application
type: Opaque
```

<h4>Brief Note on Handling Secrets</h4>


_I've covered the [process of encrypting Secrets in Kubernetes](https://operating-kubernetes.info/posts/automating-and-enabling-encryption-at-rest/) in an earlier post, but this only refers to storage on the cluster itself._

_If your threat model requires it, you might also consider making use of [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets), which allow you to securely store your `Secret` manifests as well, and check these client definitions into version control, along with other resources that do not require secrecy, without being concerned for the Secret data leaking._

_Paired with encrypted secrets in Kubernetes, this ensures the data is encrypted when sent from the client (in this case, `kubectl`) to the server, where the `Secret` is encrypted at-rest._

_I recommend, if you use a process like this in production to handle `Secret` data like credentials and certificate data, looking into the `SealedSecret` resource._

Creating the TLS Ingress
---

Once your `Secret` is defined with the certificate and key in place, you can move on to defining an `Ingress` that will use this `Secret` to provide TLS to that set of Ingress rules:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-app-tls
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
      - ${YOUR_CERTIFICATE_COMMON_NAME}
  - secretName: router-tls
  rules:
  - host: ${YOUR_CERTIFICATE_COMMON_NAME}
    http:
      paths:
      - backend:
          serviceName: web-service
          servicePort: 80
        path: /
```

So, you'll notice a couple of differences between our `Ingress` here and the one we used earlier: This one uses `annotations` to specify an Ingress class, in this case, using features provided by Nginx (TLS), and in our `tls` spec, we're defining that for our hostname (the common name for the certificate you provided), use the certificate/key pair in the `Secret` called `router-tls`, and it will apply to the rules downward in the definition, where your Ingress rules will be defined as we did above, with a `path` and `serviceName` to route to, along with the exposed ports, accessible to the specified hostname.

Additional Resources
---

[Ingress Controllers - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

[Application Load Balancing with Ingresses](https://akomljen.com/aws-alb-ingress-controller-for-kubernetes/)

[Kubernetes Ingress 101: NodePort, Load Balancers, and Ingress Controllers](https://blog.getambassador.io/kubernetes-ingress-nodeport-load-balancers-and-ingress-controllers-6e29f1c44f2d)

[kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)


