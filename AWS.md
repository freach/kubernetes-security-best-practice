# Kubernetes Security on AWS

## Host privileges :boom:

By default every Pod inherits the privileges of it's Kubernetes node. This means, if you've policies in place allowing an EC2 instance to manipulate AWS resources your Pod has the except same power to do so too.

A fix is provided by the project [kube2iam](https://github.com/jtblin/kube2iam) and [kiam](https://github.com/uswitch/kiam).

## Metadata API :boom:

Firewall access to the EC2 metdata API through which IAM role credentials can be queried. Find a fix for that issue at [https://github.com/jtblin/kube2iam#iptables](https://github.com/jtblin/kube2iam#iptables). You can check EC2 metadata API access from within a Pod by executing the following command which should fail.

```sh
curl -s http://169.254.169.254/latest/user-data
```
