# Kubernetes Security with kops

[kops](https://github.com/kubernetes/kops) is an installer which helps you create, destroy, upgrade and maintain production-grade, highly available, Kubernetes clusters on a cloud provider.
Supported providers at the time are AWS, GKE (beta support) and VMware vSphere (in alpha). Because
currently only AWS is officialy supported, this guide will refer to AWS as your cloud provider.

**kops Version 1.8.1 (git-94ef202)**

## Setup a secure cluster

### Create cluster

```sh
kops create cluster\
    --cloud aws\
    --authorization rbac\
    --topology private\
    --networking canal\
    --bastion\
    --image ami-74e6b80d\
    --master-count 3\
    --node-count 2\
    --zones eu-west-1a,eu-west-1b,eu-west-1c\
    --ssh-public-key ~/.ssh/mykey.pub\
    --state s3://k8s-mycluster-domain-tld\
    mycluster.domain.tld
```

The important bits:

* `--authorization rbac` Will set the authorization mode to RBAC *(default: AlwaysAllow)*
* `--topology private` Will use a private VPC subnet for all cluster nodes *(default: public)*
* `--networking canal` In a private topology the default networking provider `kubenet` can't be used, also we want to support network policies *(default: kubenet)*
* `--bastion` Provides an external facing host to entry the private network instances *(default: No bastion)*

**Warning: Don't apply/deploy the cluster just yet!**

### API config changes

#### Insecure Port :cloud:

It's recommended to disable the insecure (non TLS, non auth, non authz) port for the API. At the time (kops 1.8.1) the insecure port can't be closed
because of the API health check ([Issue 43784](https://github.com/kubernetes/kubernetes/issues/43784)). The port will not be exposed in the network by default and the API ELB is targeting the secure port, so it's considered OKish. From Kubernetes v1.10 the `--insecure-port` option will be deprecated.

#### Disable Profiling :cloud:

Profiling should be disabled (see: [General: API settings -> AdmissionController](README.md#api-settings)) but can't be configured through the `kubeAPIServer` setting currently. [Issue 4688](https://github.com/kubernetes/kops/issues/4688)

#### Admission Controller Config :fire:

Refer to [General: API settings -> AdmissionController](README.md#api-settings).

```yaml
spec:
  kubeAPIServer:
    admissionControl:
    - Initializers
    - NamespaceLifecycle
    - LimitRanger
    - ServiceAccount
    - PersistentVolumeLabel
    - DefaultStorageClass
    - DefaultTolerationSeconds
    - NodeRestriction
    - ResourceQuota
    - AlwaysPullImages
    - DenyEscalatingExec
```

### Deploy cluster

Apply/deploy and validated the cluster setup ...

```sh
kops update cluster $NAME --yes
kops validate cluster $NAME
```

### Add and enable PodSecurityPolicy

*Work in progress*

### Apply OS security updates :fire:

After your cluster is up, you should SSH in to the Bastion host and check each deployed EC2 instance for
security updates. Official AMIs like for example the Ubuntu AMI are relatively up to date
but AMIs in general should be considered out of date after creation. If you manage your
own AMI you should update the AMI and provide the new ID through `kops cluster edit` and reapply
the cluster, so that all instances will be recreated with the updated AMI.

## Bastion host in pre-existing cluster :cloud:

```sh
kops create instancegroup bastions --role Bastion --subnet $SUBNET
```

See: [Bastion in Kops](https://github.com/kubernetes/kops/blob/master/docs/bastion.md)

## Changing Topology of the API server :partly_sunny:

If you want to completely shutdown any public access to cluster services follow the steps for [changing the Topology of the API server](https://github.com/kubernetes/kops/blob/master/docs/topology.md#changing-topology-of-the-api-server).

**Warning:** Before changing the API to private you need some kind of VPN or similar to still be able to access the K8s API.

## Chaning DNS zone to private :partly_sunny:

By default the Route53 hosted zone managed by Kubernetes is public. This has the advantage that even with a private topology DNS records (eg. the Bastion host) can still be resolved publicly. The zone can be changed to private by passing the `--dns private` argument to `kops create cluster` or by editing the cluster after creation.

**Warning:** You need some kind of VPN or similar to still be able to resolve DNS records for your cluster zone.

## AWS specific security measures :boom:

See: [Kubernetes Security on AWS](AWS.md)
