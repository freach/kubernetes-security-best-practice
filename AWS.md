# Kubernetes Security on AWS

## Host privileges & Metadata API :boom:

By default every Pod inherits the privileges of it's Kubernetes node. This means, if you've policies in place allowing an EC2 instance to read or manipulate AWS resources your Pod has the same power to do so.

A fix is provided by the project [kube2iam](https://github.com/jtblin/kube2iam) and [kiam](https://github.com/uswitch/kiam) *(fork of kube2iam)*.

You can check AWS Metdata API access from within a Pod by executing the following command:

```sh
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials
```

This would be the most critical endpoint, but sensible information can also be retrieved through the `/user-data` endpoint or others.

```sh
curl -s http://169.254.169.254/latest/user-data
```

### kube2iam

kube2iam acts as a transparent proxy between Pods and the AWS Metdata API. An iptables rule redirects all traffic to the API to kube2iam, which proxies the traffic and blocks unprivileged access. To setup kube2iam apply the following objects to your K8s cluster.

**Note** If you're not using canal or calico as your network provider, you need to adjust `--host-interface` in `kube2iam.daemonset.yaml` accordingly, see [kub2iam iptables](https://github.com/jtblin/kube2iam#iptables).

```sh
kubectl apply -f https://raw.githubusercontent.com/freach/kubernetes-security-best-practice/master/AWS/kube2iam.serviceaccount.yaml
kubectl apply -f https://raw.githubusercontent.com/freach/kubernetes-security-best-practice/master/AWS/kube2iam.clusterrole.yaml
kubectl apply -f https://raw.githubusercontent.com/freach/kubernetes-security-best-practice/master/AWS/kube2iam.clusterrolebinding.yaml
kubectl apply -f https://raw.githubusercontent.com/freach/kubernetes-security-best-practice/master/AWS/kube2iam.daemonset.yaml
```

This is just a basic setup which will keep Pods from accessing the `/meta-data/iam/security-credentials` endpoint all other endpoints are transparently proxied and no restrictions applied. If Pods don't need AWS API access, the API should be firewalled from Pods altogether.

For a more comprehensive guide see the project's docs.
