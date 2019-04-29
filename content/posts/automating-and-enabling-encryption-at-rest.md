---
title: "Automation for, and thinking about operations for, enabling at-rest Secrets Encryption"
author: "Joseph D. Marhee"
date: 2019-04-27T09:35:58-05:00
draft: false
---
Kubernetes offers the ability to encrypt Secrets at-rest, and it's one of many features one can enable by modifying the `kube-apiserver` binary's start-up flags, for example:

```
--experimental-encryption-provider-config=/etc/kubernetes/secrets.conf
```

and this goes into the `/etc/kubernetes/manifests/kube-apiserver.yaml` file if you're a Kubeadm user. Easy enough, right? So, you just need the secrets.conf config, which just contains your encryption key (which you'll rotate periodically, and I'll cover that in a moment), and the provider for secrets encryption (these are built-in, but there are options for if you use a KMS, or your cloud provider offers an option that can be integrated with the Secrets API for this purpose--AWS Secrets Manager, for example, if you're an EKS user), so you need a file like this:

```
kind: EncryptionConfig
apiVersion: v1
resources:
- resources:
- secrets
providers:
- aescbc:
keys:
- name: key1
secret: 9cI2cvIw+wTxL0DqTIom9Ka+TUJoVJ16tl0lQ795K8I=
- identity: {}
```

which, again, seems straightforward enough, however, if your cluster spin-up routine has a high level of automation, and has integration with a config management suite, or provisioning tools like cloud-init and Terraform, you might not want to stop to configure this manually, and adding this to your spin-up routine is a consideration worth making.
In my case, I have a Terraform project that spins up and configures, via cloud-init, my cluster after the servers and network resources come online, and in this routine, if a user defines a variable like `secrets_encryption` to be `true`, a function like this runs:

```
function gen_encryption_config {
  echo "Generating EncryptionConfig for cluster..." && \
  export BASE64_STRING=$(head -c 32 /dev/urandom | base64) && \
  cat << EOF > /etc/kubernetes/secrets.conf
apiVersion: v1
kind: EncryptionConfig
resources:
- providers:
  - aescbc:
      keys:
      - name: key1
        secret: $BASE64_STRING
  resources:
  - secrets
EOF
}
```

and outputs this new configuration to `/etc/kubernetes/secrets.conf` on my cluster controller, for my kube-apiserver binary to pickup when the additional option described above is defined. In my case, my project uses Kubeadm, so I also need a function like:

```
function modify_encryption_config {
  sed -i 's|- kube-apiserver|- kube-apiserver\n    - --experimental-encryption-provider-config=/etc/kubernetes/secrets.conf|g' /etc/kubernetes/manifests/kube-apiserver.yaml && \
  sed -i 's|  volumes:|  volumes:\n  - hostPath:\n      path: /etc/kubernetes/secrets.conf\n      type: FileOrCreate\n    name: secretconfig|g' /etc/kubernetes/manifests/kube-apiserver.yaml  && \
  sed -i 's|    volumeMounts:|    volumeMounts:\n    - mountPath: /etc/kubernetes/secrets.conf\n      name: secretconfig\n      readOnly: true|g' /etc/kubernetes/manifests/kube-apiserver.yaml 
}
```

to mount the config we just generated to a volume the container running the apiserver will be able to read from, and then add the new option. 
Using most other deployment methods, you might have a systemd service managing your API server, that can be [modified like any other service](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) and then run systemctl daemon-reload and restart the service to pick up the change. This would be a good candidate for something automated on cluster spin-up. 
If you, down the road, need to rotate this encryption key, the process is something like:
Add a new entry to your `/etc/kubernetes/secrets.conf`:

```
...
  - aescbc:
      keys:
      - name: new_key
        secret: $NEW_BASE64_STRING
	  - name: key1
        secret: $BASE64_STRING
...
```

ensuring that the new key is the first in your list, preceding the old key. If you're a Kubeadm user, you'll need to open the `kube-apiserver.yaml` file to restart the apiserver process, otherwise, this will just require restarting the `apiserver` service.

2. Refresh your secrets to use the new key for encryption:

```
kubectl get secrets -o json | kubectl replace -f -
```

3. Remove the old entry, and restart the apiserver again.
Planning automation for a task like this is a little more involved than generating the configuration in the first place, or is one of few tasks you might write into a separate set of playbooks from your cluster spin-up work (i.e. you might run this via Ansible, or whatever your team uses for host management) that might not require rolling the entire cluster to bring up-to-date. 

Additional Reading
--

[Kubernetes The Hard Way, Configuring the API Server](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server)

[Twistlock - Encrypting Secrets](https://www.twistlock.com/labs-blog/kubernetes-secrets-encryption/)

[Writing a Python EncryptionConfig generator](https://medium.com/@jmarhee/example-of-yaml-generator-and-validator-in-python-5460505b5ad8)

[packet-labs/packet-multiarch-k8s-terraform - Terraform project to deploy Kubernetes on Packet](https://github.com/packet-labs/packet-multiarch-k8s-terraform)

[jmarhee/cousteau - Terraform project to deploy Kubernetes on DigitalOcean](https://github.com/jmarhee/cousteau/blob/master/controller.tpl)
