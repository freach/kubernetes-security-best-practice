# Kubernetes Security - Best Practice Guide

## Your cluster is as secure as the system running it

Before you start looking into Kubernetes security specifics you should start
with your system running Kubernetes. Go through some guides for securing your OS in general.

Here are some to begin with:

* [Red Hat Enterprise Linux 6 Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/pdf/security_guide/Red_Hat_Enterprise_Linux-6-Security_Guide-en-US.pdf)

## Private topology

If your infrastructure allows for private IP addresses you should host the cluster in a private subnet and only forward must have ports from your NAT gateway to your cluster. If you're running on a cloud provider like AWS this can be achieved through a private VPC.

### Private topology with kops

```sh
kops create cluster --topology private --networking calico ...
```

## Firewall ports

This is a general security best practice, never expose a port, which don't need exposure. IMHO defining port exposure should be done in the following order:

* Check if you can define a listen IP/interface to bind the service to, if possible 127.0.0.1/lo
* If a listen IP/interface is not possible firewall the port

Kubernetes processes like *kubelet* are opening a few ports on all network interfaces, which should be firewalled from public access. Those ports may "only" allow to query for sensitive information, but some of them allow straight full access to your cluster.

Port | Process | Description
--- | --- | ---
4149/TCP | kubelet | Default cAdvisor port used to query container metrics
10250/TCP | kubelet | API which allows full node access
10255/TCP | kubelet | Unauthenticated read-only port, allowing access to node state
10256/TCP | kube-proxy | Health check server for Kube Proxy
9099/TCP | calico-felix | Health check server for Calico (if using Calico/Canal)
6443/TCP | kube-apiserver | Kubernetes API port

Health check ports are no security threat per se by the information they expose, but critical components like the network provider could be DoSed through an exposed health check port, which would affect the hole cluster.

## Bastion host

Don't provide straight public SSH access to each Kubernetes node, use a bastion host setup where you expose SSH only on one specific host from which you SSH into all other hosts. There are quiet a few articles on how to do this, for example [https://www.nadeau.tv/ssh-with-a-bastion-host/](https://www.nadeau.tv/ssh-with-a-bastion-host/). Also SSH session recording as described in [https://aws.amazon.com/blogs/security/how-to-record-ssh-sessions-established-through-a-bastion-host/](https://aws.amazon.com/blogs/security/how-to-record-ssh-sessions-established-through-a-bastion-host/) can be useful.

For general SSH hardening check [Hardening OpenSSH](https://dev.gentoo.org/~swift/docs/security_benchmarks/openssh.html) and the OpenSSH chapter in [Applied Crypto Hardening](https://bettercrypto.org/static/applied-crypto-hardening.pdf) by bettercrypto.org.

## API authorization mode

Some installers like *kops* will use the *AlwaysAllow* authorization mode for the cluster. This would grant any authenticated entity full cluster access. Instead *RBAC* should be used for Role-based access control. To find out what your current configuration is, check the *--authorization-mode* parameter of your *kube-apiserver* processes. More information on that topic at [https://kubernetes.io/docs/admin/authorization/](https://kubernetes.io/docs/admin/authorization/).

**Note** This doesn't affect the *kubelet* authorization mode. *kubelet* itself exposes an API to execute commands through which the Kubernetes API can be bypassed completely.

## Kubelet authorization mode

*Kubelet* offers a command API used by *kube-apiserver* through which arbitrary commands can be executed on the specific node. On top of firewalling the port (10250/TCP) from public access, the *kubelet* settings *--authorization-mode=Webhook* and *--anonymous-auth=false* should be ensured.

## Cloud provider host privileges

If you're running your cluster on a cloud provider, especially AWS you must know that by default every Pod inherits the privileges of it's Kubernetes node. This means, if you've policies in place allowing an EC2 instance to manipulate AWS resources your Pod has the except same power to do so too.

## AWS Metadata API

For AWS only, you should definitely firewall access to the EC2 metdata API through which IAM role credentials can be queried. Find a fix for that issue at [https://github.com/jtblin/kube2iam#iptables](https://github.com/jtblin/kube2iam#iptables). You can check EC2 metadata API access from within a Pod by executing the following command which should fail.

```sh
curl -s http://169.254.169.254/latest/user-data
```

## Kubernetes Dashboard

Typically, the kubernetes-dashboard plugin is granted a Service Account with full cluster access to be able to see and manage all aspects of the cluster.

TBD

## Auto mount default Service Account

The *Admission Controller* ensures that all Pods have a Service Account assigned by default, which is called "default". The credentials for this Service Account will be mounted into the containers file system running in the Pod unless the auto mounting feature is disabled. The mounted token can be used to query the Kubernetes API.

```sh
kubectl patch serviceaccount default -p "automountServiceAccountToken: false"
```

This will disable the auto mounting of the Service Account token and needs to be done on a per namespace basis.

**Note** For every new namespace, the *Admission Controller* will create the *default* Service Account. Changes to this Service Account need be applied accordingly.
