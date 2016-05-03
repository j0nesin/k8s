# AWS-CoreOS
Spin up a multi-node Kubernetes cluster using CoreOS.  CoreOS provides step-by-step instructions at https://coreos.com/kubernetes/docs/latest/kubernetes-on-aws.html.  This Makefile just makes it easy to execute them all.  

The Makefile lists the env variables that need to be set before running make.  It contains helper targets to install kube-aws and create a kms-key if needed.

```
make init

make validate

make create

make destroy
```
