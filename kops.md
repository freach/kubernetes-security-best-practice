# Kubernetes Security with kops

[kops](https://github.com/kubernetes/kops) is an installer which helps you create, destroy, upgrade and maintain production-grade, highly available, Kubernetes clusters on a cloud provider.
Supported providers at the time are AWS, GKE (beta support) and VMware vSphere (in alpha). Because
currently only AWS is officialy supported, this guide will refer to AWS as your cloud provider.

**kops Version 1.8.1 (git-94ef202)**

## Setup the cluster

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

*Work in progress*

## Bastion host in pre-existing cluster

```sh
kops create instancegroup bastions --role Bastion --subnet $SUBNET
```

See: [Bastion in Kops](https://github.com/kubernetes/kops/blob/master/docs/bastion.md)

## Changing Topology of the API server :partly_sunny:

If you want to completely shutdown any public access to cluster services follow the steps for [changing the Topology of the API server](https://github.com/kubernetes/kops/blob/master/docs/topology.md#changing-topology-of-the-api-server).

**Warning** Before changing the API to private you need some kind of VPN or similar to still be able to access the K8s API.

## AWS specific security measures :boom:

See: [Kubernetes Security on AWS](AWS.md)
