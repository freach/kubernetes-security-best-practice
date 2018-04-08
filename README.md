# Kubernetes Security - Best Practice Guide

This document acts as a best practice guide to Kubernetes security. K8s is a powerful platform which can be abused in many ways if not configured properly. The authors of this guide are running Kubernetes in production and worked on several K8s projects to learn about security flaws the hard way.

* [General Security Guide](#general)
* [Cloud Provider Security Guide](#cloud-provider-guide)
* [Installer Security Guide](#installer-guide)

The severity or importance of each topic is indicated by an emoji in the topic name.

* :boom: Critical
* :fire: High
* :cloud: Medium
* :partly_sunny: Low

## General

### Your cluster is as secure as the system running it :fire:

Before you start looking into Kubernetes security specifics you should start
with your system running Kubernetes. Go through some guides for securing your OS in general.

Here are some to begin with:

* [Red Hat Enterprise Linux 6 Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/pdf/security_guide/Red_Hat_Enterprise_Linux-6-Security_Guide-en-US.pdf) *(Linux in general)*

### Private topology :partly_sunny:

If your infrastructure allows for private IP addresses you should host the cluster in a private subnet and only forward ports that are needed from the outside from your NAT gateway to your cluster. If you're running on a cloud provider like AWS this can be achieved through a private VPC.

**Why?**
In a private topology all K8s masters and nodes only have private IP addresses assigned. This greatly reduces the risk of exposing anything critical to the public by mistake or lack of knowledge. Also, experience shows that if you need to expose ports explicitly you are more conscious of the potential consequences.

### Firewall ports :fire:

This is a general security best practice: Never expose a port, which doesn't need exposure. IMHO, defining port exposure should be done in the following order:

* Check if you can define a listen IP/interface to bind the service to, if possible 127.0.0.1/lo
* If selectively binding to an IP/interface is not possible, then firewall the port

Kubernetes processes like *kubelet* are opening a few ports on all network interfaces, which should be firewalled from public access. Those ports may "only" allow to query for sensitive information, but some of them allow straight full access to your cluster.

Port | Process | Description
--- | --- | ---
4149/TCP | kubelet | Default cAdvisor port used to query container metrics
10250/TCP | kubelet | API which allows full node access
10255/TCP | kubelet | Unauthenticated read-only port, allowing access to node state
10256/TCP | kube-proxy | Health check server for Kube Proxy
9099/TCP | calico-felix | Health check server for Calico (if using Calico/Canal)
6443/TCP | kube-apiserver | Kubernetes API port

Health check ports are no security threat _per se_ stemming from the information they expose, but critical components like the network provider could be DoSed through an exposed health check port, which would affect the whole cluster. Additionally, unknown exploits could potentially endanger security.

**Why?** Every port exposed to a network poses a potential attack vector. To minimize risk, exposure should be avoided if possible.

### Bastion host :cloud:

Don't provide straight public SSH access to each Kubernetes node, use a bastion host setup where you expose SSH only on one specific host from which you SSH into all other hosts. There are quite a few articles on how to do this, for example [https://www.nadeau.tv/ssh-with-a-bastion-host/](https://www.nadeau.tv/ssh-with-a-bastion-host/). Also SSH session recording as described in [https://aws.amazon.com/blogs/security/how-to-record-ssh-sessions-established-through-a-bastion-host/](https://aws.amazon.com/blogs/security/how-to-record-ssh-sessions-established-through-a-bastion-host/) can be useful.

For general SSH hardening check [Hardening OpenSSH](https://dev.gentoo.org/~swift/docs/security_benchmarks/openssh.html) and the OpenSSH chapter in [Applied Crypto Hardening](https://bettercrypto.org/static/applied-crypto-hardening.pdf) by bettercrypto.org.

**Why?** SSH is a critical service which is under constant attack if exposed to the public. Still today, key-based authentication is not used consistently and everywhere and dictionary attacks or exploits will eventually lead to intrusion. To minimize risk a hardened bastion host is introduced and SSH blocked for public access on all other nodes.

### Kubernetes Security Scan with kube-bench :fire:

A very helpful tool to eliminate roughly 95% of the configuration flaws is [kube-bench](https://github.com/aquasecurity/kube-bench). A master or a node and their control-plane components are checked by applying the [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/) which results in specific guidelines to secure your cluster setup. This should be a first step before going through any specific Kubernetes security issues or security enhancements.

### API authorization mode & anonymous auth :boom:

Some installers like *kops* will use the *AlwaysAllow* authorization mode for the cluster. This would grant any authenticated entity full cluster access. Instead, *RBAC* should be used for role-based access control. To find out what your current configuration is, check the *--authorization-mode* parameter of your *kube-apiserver* processes. More information on that topic at [https://kubernetes.io/docs/admin/authorization/](https://kubernetes.io/docs/admin/authorization/). To enforce authentication, make sure anonymous auth is disabled by setting *--anonymous-auth=false*.

**Note** This doesn't affect the *kubelet* authorization mode. The *kubelet* itself exposes an API to execute commands through which the Kubernetes API can be bypassed completely.

### Kubelet authorization mode & anonymous auth :boom:

The *kubelet* offers a command API used by *kube-apiserver* through which arbitrary commands can be executed on the specific node. On top of firewalling the port (10250/TCP) from public access, the *kubelet* settings *--authorization-mode=Webhook* and *--anonymous-auth=false* should be ensured.

## Auto mount default Service Account :partly_sunny:

The *Admission Controller* ensures that all Pods have a Service Account assigned by default, which is called "default". The credentials for this Service Account will be mounted into the containers file system running in the Pod unless the auto mounting feature is disabled. The mounted token can be used to query the Kubernetes API.

```sh
kubectl patch serviceaccount default -p "automountServiceAccountToken: false"
```

This will disable the auto mounting of the Service Account token and needs to be done on a per namespace basis.

**Note** For every new namespace, the *Admission Controller* will create the *default* Service Account. Changes to this Service Account need be applied accordingly.

### Use Network Policies :cloud:

[Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) are firewall rules for Kubernetes. If you're using a network provider which supports Network Policies, you should definitely use them to secure internal cluster communication and external cluster access. By default, there are no restrictions in place to limit pods from communicating with each other.

Check [Kubernetes Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes) for an awesome starting point. If your network provider doesn't support network policies, consider switching to one which does, check [https://kubernetes.io/docs/concepts/cluster-administration/networking/](https://kubernetes.io/docs/concepts/cluster-administration/networking/).

### Restrict "docker image pull" :fire:

Docker images are a completely uncontrolled environment. Everyone with access to the Docker socket or Kubernetes API can pull any image they like. Because of that many Kubernetes clusters secretly became Bitcoin miners, because of infected Docker images or Kubernetes security issues. The Docker plugin [Docker Image policy plugin](https://github.com/freach/docker-image-policy-plugin) will help you with that problem. The plugin hooks into the internal Docker API and enforces a set of black and white list rules to restrict what images can be pulled.

Ultimately Docker is pulling an image, so securing Docker is considered a good approach but alternatively Kubernetes also provides a way. The *AdmissionController* provides the [ImagePolicyWebhook](https://kubernetes.io/docs/admin/admission-controllers/#imagepolicywebhook) through which a provided web service can intercept image pulls.

### Kubernetes Dashboard :boom:

Prio to version 1.8.0, the `kubernetes-dashboard` plugin was granted a Service Account with full cluster access to be able to see and manage all aspects of the cluster.

Verify that there is no ClusterRolebinding to `cluster-admin` left behind. Otherwise clicking `SKIP` on the sign-in page will grant full access.

`kubectl -n kube-system get clusterrolebinding kubernetes-dashboard -o yaml`

By default the dashboard is not exposed to the public Internet and it should be avoided to change that. Reasons why we could see with the [Tesla hack discovered by RedLock](https://blog.redlock.io/cryptojacking-tesla).

### Securing a Cluster (by Kubernetes project)

The Kubernetes project itself also has some notes on how to secure a cluster. See the [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/) chapter from the Kubernetes docs.

## Cloud Provider Guide

* [AWS](AWS.md)

## Installer Guide

* [kops](kops.md)

## Author

* [Simon Pirschel](https://aboutsimon.com/) - Kubernetes Consultant & DevOps Specialist

Thanks to all the contributors to this guide assuring good quality.
