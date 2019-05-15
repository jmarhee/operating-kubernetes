---
title: "Managing Kubeadm Cluster Operations With Terraform"
date: 2019-05-15T10:09:12-05:00
author: "Joseph D. Marhee"
draft: false
---

Part of operating Kubernetes in production, much like any other orchestration solution, is figuring out how it fits into things like your organization's use of configuration management, automation tooling, CI/CD pipelines, etc. and a common one is how it can work to your advantage with a tool like [Terraform](terraform.io). 

What I'm talking about here is less of a Kubernetes operations example, but more of an approach to executing a common administration task in Kubernetes using popular DevOps tools like Terraform, and the Kubeadm cluster manager tools. So, while a departure from the operations deep dives, this may help you better understand the benefits of these third-party tools on operating a Kubernetes cluster, but also how to integrate with existing solutions you may, or may not, be using already in production. 

I've put together a few projects for provisioning Kubernetes clusters, but I'll highlight this one in particular here for a reason I'll go into in a moment:

[Multi-Architecture Kubernetes on Packet](https://github.com/packet-labs/packet-multiarch-k8s-terraform)

**Disclosure**: I am current an Ecosystem Engineer at [Packet](packet.com), but this will only tangentially impact the usefulness of the above, and at the end of this piece, I'll link guides for similar implementations I've put together that are more platform-agnostic. 

The reason I highlight this project is that it take using a Kubernetes installation tool like Kubeadm, and makes its components modular; in this case, the tool manages three pieces of state exceptionally modularly out of the box:

- The Kubernetes cluster controller.
- Your Kubernetes nodes.
- The Kubeadm token, which authenticates a node to join a cluster.

The terraform project above provisions a cluster by generating a token, initializing a controller with that token, and provisions a pool of nodes using that token. Terraform enters the picture by creating a Terraform [module](https://www.terraform.io/docs/configuration/modules.html), for two of those components, the token, and the node pool (not individual nodes), into your Terraform state to make tasks like migrating workloads between pools, and scaling a cluster, trackable pieces of state along with the rest of your infrastructure in Terraform.

So, in the above repo, in `3-kube-node.tf`, you'll see the base node pool module, `node_pool_blue`:

```
module "node_pool_blue" {
  source = "modules/node_pool"

  kube_token         = "${module.kube_token_1.token}"
  kubernetes_version = "${var.kubernetes_version}"
  pool_label         = "blue"
  count_x86          = "${var.count_x86}"
  count_arm          = "${var.count_arm}"
  plan_x86           = "${var.plan_x86}"
  plan_arm           = "${var.plan_arm}"
  facility           = "${var.facility}"
  cluster_name       = "${var.cluster_name}"
  controller_address = "${packet_device.k8s_primary.network.0.address}"
  project_id         = "${var.project_id}"
}
```

a lot of this won't be immediately relevant (i.e. Packet-specific variables like `project_id`, etc.), but the important lines are `kube_token` and `pool_label`, these will change as you add pools, and variables like `count_x86` (the number of x86 nodes in that pool), will scale up and down to operate that module's node pool. 

Let's assume you've `terraform apply`'d this project as-is to create a cluster, with a `count_x86` of `3` and a `count_arm` (number of ARM node) of `1`, to create `node_pool_blue` of 4 nodes total. Now, you want to create a new pool, consisting of the same number of nodes of each architecture, but with a new `kubernetes_version`, so you create a new instance of that module, `node_pool_green`:

```
module "node_pool_green" {
  source = "modules/node_pool"

  kube_token         = "${module.kube_token_2.token}"
  kubernetes_version = "${var.kubernetes_version}"
  pool_label         = "green"
  count_x86          = "${var.count_x86}"
  count_arm          = "${var.count_arm}"
  plan_x86           = "${var.plan_x86}"
  plan_arm           = "${var.plan_arm}"
  facility           = "${var.facility}"
  cluster_name       = "${var.cluster_name}"
  controller_address = "${packet_device.k8s_primary.network.0.address}"
  project_id         = "${var.project_id}"
}
```

You'll notice, all I did was copy and paste the module, update `pool_label` to `green`, and I'll have updated `kube_token` to reference `kube_token_2`, which means I'll need a new token module instance as well, so back in `1-provider.tf`, I'll add:

```
module "kube_token_2" {
  source = "modules/kube-token"
}
``` 

apply that change:

```
terraform apply -target=module.kube_token_2
```

and register that token with the cluster controller:

```
TOKEN2="$(terraform state show module.kube_token_1.random_string.kube_init_token_a | grep result | awk '{print $3}')$(terraform state show module.kube_token_1.random_string.kube_init_token_b | grep result | awk '{print $3}')" ; \
ssh root@${CONTROLLER_IP} kubeadm token create $TOKEN2
```

so once I apply the new module for the `green` `node_pool`:

```
terraform apply -target=module.node_pool_green
```

I'll have two pools of 4 nodes, with the new pool using this new token (checked into Terraform's state, and now registered on the controller), and you'll see them join the cluster while monitoring `kubectl get nodes -w` as they come online and cloud-init completes on the provider-side.

This allows you to do an operation like [`cordon`](https://www.mankier.com/1/kubectl-cordon) and [`drain`](https://www.mankier.com/1/kubectl-drain) the nodes in `pool_blue` (thus rescheduling them on the new nodes), and once completed, set `count_x86` and `count_arm` to `0` in Terraform, and apply to destroy the old pool:

```
terraform apply -target=module.node_pool_blue
```

This is a very crude flow, but it introduces some level of automation, and statefulness to your infrastructure rollout process, and if you use Terraform regularly, it can be an effective cost controller for your infrastructure provider usage. The primary benefit of beginning to operate your cluster (even if, for example, changes to your Terraform repo are planned and applied by CI/CD tools on merges to a master branch), is that it makes these changes to your cluster testable, and injects a measure of checking your work with automation, and keeping an eye on what your automation is doing to your infrastructure by keeping state checked in this manner. 

Additional Reading
---

[packet-labs/packet-k3s](https://github.com/packet-labs/packet-k3s)

[packet-labs/openstack-kubernetes-tf](https://github.com/packet-labs/openstack-kubernetes-tf)

[jmarhee/cousteau](https://github.com/jmarhee/cousteau)

[Automating Node Pools with Terraform](https://dev.to/jmarhee/automating-kubernetes-node-pools-with-terraform-1e5l)

[Jmarhee on Dev.to - Writing Reusable Terraform Modules](https://www.nearform.com/blog/writing-reusable-terraform-modules/)

[Kubernetes Blog - Safely Drain a Node while Respecting SLO](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
