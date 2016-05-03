# The env variables must be defined externally

# AWS_KEYPAIR=<keypair-name>
# AWS_REGION=<region>
# K8S_CLUSTER_NAME=<cluster-name>
# K8S_CLUSTER_DNS=<dns-name>
# AWS_KMS_KEY=<arn:aws:kms:<region>:xxxxx:key/xxxx>

KUBE_AWS_VERSION = 0.6.1

# get the kube-aws executable
kube-aws:
	gpg2 --keyserver pgp.mit.edu --recv-key FC8A365E
	gpg2 --fingerprint FC8A365E
	curl -L -O https://github.com/coreos/coreos-kubernetes/releases/download/v$(KUBE_AWS_VERSION)/kube-aws-darwin-amd64.tar.gz
	curl -L -O https://github.com/coreos/coreos-kubernetes/releases/download/v$(KUBE_AWS_VERSION)/kube-aws-darwin-amd64.tar.gz.sig
	gpg2 --verify kube-aws-darwin-amd64.tar.gz.sig kube-aws-darwin-amd64.tar.gz
	tar zxvf kube-aws-darwin-amd64.tar.gz
	rm kube-aws-darwin-amd64.tar.gz*

# This only needs to be run once for any given region.  Update the AWS_KMS_KEY env var with its resulting arn.
kms-key:
	aws kms --region=$(AWS_REGION) create-key --description="kube-aws assets"

# generate the config files for building the AWS environment and render them for inspection
# including the CloudFormation config
init: clean
	# generate cluster.yaml
	./darwin-amd64/kube-aws init --cluster-name=$(K8S_CLUSTER_NAME) \
	--external-dns-name=$(K8S_CLUSTER_DNS) \
	--region=$(AWS_REGION) \
	--availability-zone=$(AWS_REGION)a \
	--key-name=$(AWS_KEYPAIR) \
	--kms-key-arn="$(AWS_KMS_KEY)"
	# render credentials and config files
	./darwin-amd64/kube-aws render
	# render AWS cloudformation config for inspection (optional)
	./darwin-amd64/kube-aws up --export

clean:
	rm -rf credentials userdata cluster.yaml kubeconfig *stack-template.json

validate:
	./darwin-amd64/kube-aws validate

# create the AWS environment by creating a CloudFormation stack
create:
	./darwin-amd64/kube-aws up
	@echo ""
	@echo "You need to map the Kube Controller IP address to the DNS name $(K8S_CLUSTER_NAME) in /etc/hosts or in your DNS provider as an alias"
	@echo ""

# destroy the AWS environment
destroy:
	./darwin-amd64/kube-aws destroy

# get the cluster controller externl ip for the created stack.  Needs to alias to the external-dns-name.
cluster-ip:
	aws ec2 describe-instances --filters "Name=tag:Name,Values=kube-aws-controller" | grep PublicIpAddress

test:
	kubectl --kubeconfig=kubeconfig get nodes
