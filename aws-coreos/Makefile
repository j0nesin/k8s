# Deploy Kubernetes cluster base on CoreOS install instructions

# The env variables must be defined externally

# AWS_ACCESS_KEY_ID=<key>
# AWS_SECRET_ACCESS_KEY=<key>
# AWS_DEFAULT_REGION=<region>
# AWS_KEYPAIR=<keypair-name>
# K8S_CLUSTER_NAME=<cluster-name>
# K8S_CLUSTER_DNS=<dns-name>
# AWS_KMS_KEY=<arn:aws:kms:<region>:xxxxx:key/xxxx>

# find latest version at https://github.com/coreos/coreos-kubernetes/releases
KUBE_AWS_VERSION ?= 0.6.1
PLATFORM ?= linux-amd64
#PLATFORM ?= darwin-amd64

.PHONY:	kubeconfig dashboard-add-on

# get the kube-aws executable
kube-aws:
	gpg2 --keyserver pgp.mit.edu --recv-key FC8A365E
	gpg2 --fingerprint FC8A365E
	curl -L -O https://github.com/coreos/coreos-kubernetes/releases/download/v$(KUBE_AWS_VERSION)/kube-aws-$(PLATFORM).tar.gz
	curl -L -O https://github.com/coreos/coreos-kubernetes/releases/download/v$(KUBE_AWS_VERSION)/kube-aws-$(PLATFORM).tar.gz.sig
	gpg2 --verify kube-aws-$(PLATFORM).tar.gz.sig kube-aws-$(PLATFORM).tar.gz
	tar zxvf kube-aws-$(PLATFORM).tar.gz
	rm kube-aws-$(PLATFORM).tar.gz*

kube-aws-no-sig:
	curl -L -O https://github.com/coreos/coreos-kubernetes/releases/download/v$(KUBE_AWS_VERSION)/kube-aws-$(PLATFORM).tar.gz
	tar zxvf kube-aws-$(PLATFORM).tar.gz
	rm kube-aws-$(PLATFORM).tar.gz*

# This only needs to be run once for any given region.  Update the AWS_KMS_KEY env var with its resulting arn.
kms-key:
	aws kms --region=$(AWS_DEFAULT_REGION) create-key --description="kube-aws assets"

# generate the config files for building the AWS environment and render them for inspection
# including the CloudFormation config
init:
	# generate cluster.yaml
	./$(PLATFORM)/kube-aws init --cluster-name=$(K8S_CLUSTER_NAME) \
	--external-dns-name=$(K8S_CLUSTER_DNS) \
	--region=$(AWS_DEFAULT_REGION) \
	--availability-zone=$(AWS_DEFAULT_REGION)a \
	--key-name=$(AWS_KEYPAIR) \
	--kms-key-arn="$(AWS_KMS_KEY)"
	# render credentials and config files
	./$(PLATFORM)/kube-aws render
	# render AWS cloudformation config for inspection (optional)
	./$(PLATFORM)/kube-aws up --export

validate:
	./$(PLATFORM)/kube-aws validate

# create the AWS environment by creating a CloudFormation stack
create:
	./$(PLATFORM)/kube-aws up
	@echo ""
	@echo "You need to map the Kube Controller IP address to the DNS name $(K8S_CLUSTER_NAME) in /etc/hosts or in your DNS provider as an alias"
	@echo ""

# set the kube config parameters for this cluster and set it to current default
kubeconfig:
	kubectl config set-cluster $(K8S_CLUSTER_NAME) --server=https://$(K8S_CLUSTER_DNS) --certificate-authority=$(PWD)/credentials/ca.pem
	kubectl config set-credentials $(K8S_CLUSTER_NAME) --certificate-authority=$(PWD)/credentials/ca.pem --client-key=$(PWD)/credentials/admin-key.pem --client-certificate=$(PWD)/credentials/admin.pem
	kubectl config set-context $(K8S_CLUSTER_NAME) --cluster=$(K8S_CLUSTER_NAME) --user=$(K8S_CLUSTER_NAME)
	kubectl config use-context $(K8S_CLUSTER_NAME)

# destroy the AWS environment
destroy:
	./$(PLATFORM)/kube-aws destroy
	rm -rf credentials userdata cluster.yaml kubeconfig *stack-template.json

# get the cluster controller externl ip for the created stack.  Needs to alias to the external-dns-name.
cluster-ip:
	aws ec2 describe-instances --filters "Name=tag:Name,Values=kube-aws-controller" | grep PublicIpAddress

test:
	kubectl --kubeconfig=kubeconfig get nodes
