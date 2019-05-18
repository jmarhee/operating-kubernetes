---
title: "Using InitContainers to pre-populate Volume data in Kubernetes"
date: 2019-05-17T15:48:49-05:00
author: "Joseph D. Marhee"
draft: false
---

**Originally published in posts on [Medium](https://medium.com/@jmarhee/using-initcontainers-to-pre-populate-volume-data-in-kubernetes-99f628cd4519), and [Dev.to](https://dev.to/jmarhee/getting-started-with-kubernetes-initcontainers-and-volume-pre-population-j83)**

The InitContainer resource in Kubernetes can be used to run tasks before the rest of a pod is deployed. A common use-case is to pre-populate config files (one notable example is for providing a custom metrics endpoint), but basically can be used to complete any kind of container-based task before the rest of the pod containers are brought online.

Let’s take the example of a pod like this:

```
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:latest
```
Suppose that the my-app image is stateless, but requires a parameter to the component out-of-cluster that requires modification to a configuration file (let’s call this /data/config.json ) before it will function as expected.

This is where an InitContainer can be useful. Your Pod spec can be modified to use the InitContainer resource to build the pod to provide a JSON body like:

```
{
  "address":"database:port/whatever"
}
```
into a file like /data/config.json using common Kubernetes resources like volumes :

```
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:latest
    volumeMounts:
    - name: config-data
      mountPath: /data
  initContainers:
  - name: config-data
    image: busybox
    command: ["echo","-n","{'address':'10.0.1.192:2379/db'}", ">","/data/config"]
    volumeMounts:
    - name: config-data
      mountPath: /data
  volumes:
  - name: config-data
    hostPath: {}
```
So, you’ll notice 3 changes here:

The addition of a volume (using the hostPath , but this can use any volume type you’d like) that will provide a way to share data between the InitContainer and the rest of the pod.
The addition of a volumeMount to both the InitContainer and the pod Containers, allowing the pod containers to read the data created by the init task.
The InitContainer spec itself, which will be the work done before the rest of the pod comes online (in this case, just echo-ing a JSON body into the hostPath non-persistent volume).
This will allow you to provide pre-preparation to whatever data you’d like the pods eventually running to be exposed to.

Earlier, I mentioned custom metrics, and an example of where this becomes useful is with HorizontalPodAutoscaler objects, where the Pod IP has to be written to a config in order for the metrics to be read from the Pod by Heapster:

Kubernetes autoscaling based on custom metrics without using a host port
How to set up horizontal pod autoscaling based on application-provided custom metrics on minikubemedium.com
What this does is, for example, allow your app to write a custom metric to a URI, which Heapster can then read, and expose to Kubernetes for a potential scaling event.

In the event of such a configuration, you might have an app like:
```
from flask import Flask
import time

INTERVAL=5
seconds = []

def qps():

	if len(seconds) > INTERVAL:
		del seconds[:%s % (INTERVAL + 1)]

	seconds.append(time.strftime("%H%M%S"))

	avgRps = len(seconds) / len(set(seconds))

	return "qps %s" % (avgRps)

app = Flask(__name__)

@app.route("/metrics")
def metrics():
	rp5s = qps()
	return rp5s

app.run(host='0.0.0.0')
```
so you’d like custom-metrics to be checked against ${POD_IP}:5000/metrics to return the average requests per second over an interval of the previous 5 seconds; something that is not provided by Heapster (which does not collect, or provide as an autoscaleable metric), but by the application itself. Which means, you could use an InitContainer to run a command like:
`"command": ["sh", "-c", "echo \"{\\\"endpoint\\\": \\\"http://$POD_IP:5000/metrics\\\"}\" > /etc/custom-metrics/definition.json"]`,
and create a volumeMount to that effect before creating your HorizontalPodAutoscaler.

Let's take a look at another example with a slightly different use case for InitContainer resources, where rather than mounting a config (something we can do using a couple of different strategies, ConfigMaps, etc.), we're going to assume each Pod relaunch requires fresh data from an external resource. In many cases, you'll see it used to do things like prepopulate data in a volume before creating the containers in a `Pod` deployment, so upon spinning up the containers, the volume data is already initialized.

In my case for this second example, I have a simple web frontend with a single static page, and it boots using the standard `nginx` Docker image to serve a manifest file:

```
FROM nginx

COPY index.html /usr/share/nginx/html/index.html
COPY smartos.ipxe /usr/share/nginx/html/smartos.ipxe
```

This image builds and downloads very quickly, which is awesome, but partly because it's stateless; the the `smartos.ipxe` file, for example, makes reference to release data that, upon spinning up the app, needs to be available or else those references will not work as intended (abstracted as a `404` HTTP response):

```
#!ipxe
dhcp
set base-url http://sdc-ipxe.east.gourmet.yoga
kernel ${base-url}/smartos/smartos/platform/i86pc/kernel/amd64/unix -B smartos=true,console=ttyb,ttyb-mode="115200,8,n,1,-"
module ${base-url}/smartos/smartos/platform/i86pc/amd64/boot_archive type=rootfs name=ramdisk
boot
```

These files, however, are not part of the app image because, for example, it updates frequently, so each time we rollout a new version, we'd like the latest release available in the volume, and since we don't need to maintain these files in the image, or else it would be very large and costly to store in our registry, we can mount a Volume to the containers in the Pod to provide them.

So, basically, we need a way to pre-populate a volume we're mounting to `/usr/share/nginx/html/smartos`.

Using the `InitContainer` resource, we can specify a command to be run, and like any other container in a `Pod`, we can allocate a volume to be mounted, so let's start with a Kubernetes manifest like this:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sdc-ipxe-deployment
  labels:
    app: sdc-ipxe
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sdc-ipxe
  template:
    metadata:
      labels:
        app: sdc-ipxe
    spec:
      initContainers:
      - name: config-data
        image: ubuntu:xenial
        command: ["/bin/sh","-c"]
        args: ["apt update; apt install -y wget tar; wget https://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/platform-latest.tgz; tar xvf platform-latest.tgz -C /data; mkdir /data/smartos; mv /data/platform* /data/smartos/platform"]
        volumeMounts:
        - mountPath: /data
          name: sdc-data
      volumes:
      - name: sdc-data
        hostPath:
          path: /mnt/kube-data/sdc-ipxe/
          type: DirectoryOrCreate

```

So, at this point, we're provisioning the volume `sdc-data`, mounting it to an `initContainer` called `config-data` at `/data`, and running:

```
"apt update; apt install -y wget tar; wget https://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/platform-latest.tgz; tar xvf platform-latest.tgz -C /data; mkdir /data/smartos; mv /data/platform\* /data/smartos/platform"
```

to download and extract the data to the volume. Now, we we add a `containers:` key to the manifest, and attach that Volume again, the prepopulated data will be availabnle:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sdc-ipxe-deployment
  labels:
    app: sdc-ipxe
...
      containers:
      - name: sdc-ipxe
        image: coolregistryusa.bix/jmarhee/sdc-ipxe:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /usr/share/nginx/html/smartos
          name: sdc-data
...
```

with the volume mounted where the app in the container expects, `/usr/share/nginx/html/smartos` using the same `sdc-data` volume.

Other examples of this pattern being useful can be if your application relies on configurations with variable requirements; maybe you're issued a token, or an address is fairly dynamic, and this needs to be passed through files on disk, rather than the environment (i.e. a loadbalancer, or webserver, or database client with a configuration file, not easily handled through a Secret or ConfigMap because of how often they change), this provides a somewhat easily programmable interface for pre-populating or completing the templating of data handed to a container.


