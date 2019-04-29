---
title: "Quick Intro to Jobs and Cronjobs and What to Do if They Fail"
date: 2019-04-28T10:30:31-05:00
draft: false
---

One of the finer controls in Kubernetes is over container lifecycle; consider
that if a Pod is your basic deployment unit, and replication controllers like 
Deployments  and DaemonSets  are just reconciliation loops that track the
desired number of replicas against the running number of replicas, why shouldn't
you be able to schedule a single-use, or recurring one-off executions of a
script in a Pod  as well? Well, that's exactly what the Job  resource is for in
Kubernetes!

The primary difference between Job  and CronJob  workloads is exactly what it
sounds like; a CronJob  will run on a schedule, rather than as a one-off task,
and you can mix these workloads, as you'll see later on. I'll focus mostly on 
CronJobs  in this piece, but will provide some examples for Job  workloads as
well.

Consider the following example: I needed a script to periodically pan a
directory (mounted as a Volume) of videos, take a screenshot, and store that
screenshot in another volume. So, I created a bash one-liner to do that, and
created a Docker image that uses that command as an ENTRYPOINT
[https://blog.codeship.com/understanding-dockers-cmd-and-entrypoint-instructions/] 
 (the command the container runs on boot, and then exits upon completion).

It doesn't make sense to run this as a persistent service; I can't use restart
policies, for example, to say I'd like this to happen every 6 hours, and I don't
want to keep the resources required for this locked up when it's not in-use, so
I wrote a CronJob:

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: capture-create
spec:
  schedule: "0 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: capture-create
            image: jmarhee/capture-create:latest
            env:
            - name: LNVC_DATA_PATH
              value: /media
            - name: LNVC_PREV_PATH
              value: /captures
            volumeMounts:
            - name: lnvc-content-volume
              mountPath: /media
            - name: lnvc-caps-volume
              mountPath: /captures
          volumes:
          - name: lnvc-content-volume
            hostPath:
              path: /mnt/volume_tor1_01/kube-data/lnvc-data
              type: Directory
          - name: lnvc-caps-volume
            hostPath:
              path: /mnt/volume_tor1_01/kube-data/images
              type: Directory
          restartPolicy: OnFailure
          imagePullSecrets:
          - name: myregistrykey
          nodeSelector:
            biggie-storage: "true
```

So, you'll see in the above, the definition looks a lot  like a normal Pod 
definition, and that's because it is; you're defining things like volumeMounts 
for storage, env  for your variables, and even nodeSelector  to tell the API
where to schedule these containers (in my case, biggie-storage  nodes have
appropriate space and performance constraints for this job!). The schedule  key
takes your normal cron syntax [crontab.guru]  as a value, and runs the job on
that schedule.

When you run kubectl get pods  you'll see the pods as you would for normal
workloads, and you'll also see that the API is cleaning up after these jobs once
they exit:

```
root@krebstar-primary:~# kubectl get pods -l job=capture
capture-create-1556492400-nchfh               0/1     Completed   0          3h3m
capture-create-1556496000-tkfxz               0/1     Completed   0          123m
capture-create-1556499600-jdh65               0/1     Completed   0          63m
capture-create-1556503200-4njrv               1/1     Running     0          3m16s
```

and you'll see statuses like Completed  if the job ran and exited 0, Running 
for jobs in progress, and Error  or whatever failed state for jobs that did not
run successfully, and troubleshooting them is similar to any other workload:

```
kubectl describe pod $POD_NAME
```

to check the events for that Pod  or checking the container workload itself:

```
kubectl logs $POD_NAME [$CONTAINER if required]
```

If you'd like to run the job manually, as a one-off task before the next cron
run (if the restart policy does not catch the failure, or you'd like to make a
change to the job on the spec-level), there is also a mechanism in kubectl  to
provide for this:

If you have a scheduled CronJob  in Kubernetes that has failed, or has no Pods
available to complete the task, you can run the task as a Job instead to
complete the task.

To create it from the manifest in Kubernetes for the existing CronJob, you can
use the from argument with kubectl to define the source job to be templated for
the new Job:

```
kubectl create job my-job --from=cronjob/my-cron-job
```

which will proceed to schedule the pods for this task. You can modify this Job,
after mounting the spec from the failed CronJob, and run this task out of scope
for the recurring task, for example, with the edit  argument, and then treat
this as an at-will workload. 

Further Examples & Reading
Analyzing Your On-Call Activity with Kubernetes Jobs
[https://dev.to/jmarhee/analyzing-your-on-call-activity-with-kubernetes-jobs-4559]

MongoDB Backups with Kubernetes Jobs
[https://dev.to/jmarhee/mongodb-backups-with-kubernetes-jobs-2n62]

Kubernetes By Example - Jobs [http://kubernetesbyexample.com/jobs/]
