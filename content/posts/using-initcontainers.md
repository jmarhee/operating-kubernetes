---
title: "Using InitContainers to pre-populate Volume data in Kubernetes"
date: 2019-05-17T15:48:49-05:00
author: "Joseph D. Marhee"
draft: false
---

**Originally published on [Medium](https://medium.com/@jmarhee/using-initcontainers-to-pre-populate-volume-data-in-kubernetes-99f628cd4519)**

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
