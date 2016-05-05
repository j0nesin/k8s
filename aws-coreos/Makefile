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
PROXY_PORT ?= 8011

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
	@echo ""
	@echo "WARNING : Required steps to use ES Logging add-on :"
	@echo "CoreOS has a bug that prevents fluentd from accessing the logs on each node."
	@echo "See https://github.com/coreos/coreos-kubernetes/issues/322 for issue details."
	@echo "Modify the cloud-config files generated in /userdata as follows to resolve the issue:"
	@echo "1.) Update the RKT_OPS line in each file to include the following parameters:"
	@echo "		--volume container,kind=host,source=/var/log/containers,readOnly=false --mount volume=container,target=/var/log/containers"
	@echo "2.) Add the following section under write_files:"
	@echo "  - path: /var/log/containers/dummy.txt"
	@echo "			content: |"
	@echo "				dummy file to generate directory /var/log/containers"
	@echo ""
	@echo "WARNING : Required step to use Influxdb-Grafana Monitoring add-on"
	@echo "The default Heapster definition must be removed.  It will be replaced when the add-on is deployed."
	@echo "1.) Remove the section in userdata/cloud-config-controller that begins with:"
	@echo "    - path: /srv/kubernetes/manifests/heapster-dc.json"
	@echo ""

validate:
	./$(PLATFORM)/kube-aws validate

# create the AWS environment by creating a CloudFormation stack
create:
	./$(PLATFORM)/kube-aws up
	@echo ""
	@echo "You need to map the Kube Controller IP address to the DNS name '$(K8S_CLUSTER_DNS)' in /etc/hosts or in your DNS provider as an alias"
	@echo ""
	@echo "ssh to boxes as core@<ipaddress> or core@$(K8S_CLUSTER_DNS)"
	@echo ""

# get the cluster controller externl ip for the created stack.  Needs to alias to the external-dns-name.
cluster-ip:
	aws ec2 describe-instances --filters "Name=tag:Name,Values=kube-aws-controller" | grep PublicIpAddress

# add the kube config parameters for this cluster to the global configs and set it to active context
kubeconfig:
	kubectl config set-cluster $(K8S_CLUSTER_NAME) --server=https://$(K8S_CLUSTER_DNS) --certificate-authority=$(PWD)/credentials/ca.pem
	kubectl config set-credentials $(K8S_CLUSTER_NAME) --certificate-authority=$(PWD)/credentials/ca.pem --client-key=$(PWD)/credentials/admin-key.pem --client-certificate=$(PWD)/credentials/admin.pem
	kubectl config set-context $(K8S_CLUSTER_NAME) --cluster=$(K8S_CLUSTER_NAME) --user=$(K8S_CLUSTER_NAME)
	kubectl config use-context $(K8S_CLUSTER_NAME)

proxy: kubeconfig
	kubectl proxy --port $(PROXY_PORT)
	@echo "Proxy created at http://localhost:$(PROXY_PORT)"

test:
	kubectl --kubeconfig=kubeconfig get nodes

# destroy the AWS environment
destroy:
	./$(PLATFORM)/kube-aws destroy

clean:
	rm -rf credentials userdata cluster.yaml kubeconfig *stack-template.json

####### Add-Ons #######

# get kubeernetes src to launch add-ons
K8S_SRC = ./kubernetes
k8s-src:
	git clone --depth 1 https://github.com/kubernetes/kubernetes.git
# update src
update-k8s-src:
	cd $(K8S_SRC) && git pull

dashboard: kubeconfig
	# deploy all the RCs and SVCs in the add-on directory
	kubectl create -f $(K8S_SRC)/cluster/addons/dashboard/
	@echo ""
	@echo "Access dashboard at http://localhost:$(PROXY_PORT)/ui"
	@echo ""

delete-dashboard: kubeconfig
	kubectl delete --ignore-not-found -f $(K8S_SRC)/cluster/addons/dashboard/

es-logging:
	# deploy all the RCs and SVCs in the add-on directory
	kubectl create -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/
	# need fluentd-elasticsearch container to run on all nodes.  Two options for this:
	# - Copy $(K8S_SRC)/cluster/saltbase/salt/fluentd-es/fluentd-es.yaml to /etc/kubernetes/manifests on all machines
	# - Deploy container as daemon-set pod using custom file.  Keep custom file versions updated with K8S src file listed above
	# Here are are doing the latter
	kubectl create -f ./fluentd-es-daemonset.yaml
	@echo ""
	@echo "Test ES : http://localhost:$(PROXY_PORT)/api/v1/proxy/namespaces/kube-system/services/elasticsearch-logging/_cat/indices?v"
	@echo "Test ES : http://localhost:$(PROXY_PORT)/api/v1/proxy/namespaces/kube-system/services/elasticsearch-logging/_search?pretty=true"
	@echo ""
	@echo "Kibana : http://localhost:$(PROXY_PORT)/api/v1/proxy/namespaces/kube-system/services/kibana-logging/"
	@echo "You will be presented with a page that asks you to configure your view of the ingested logs. Select the option for timeseries values and select @timestamp. On the following page select the Discover tab and then you should be able to see the ingested logs."
	@echo ""

delete-es-logging:
	kubectl delete --ignore-not-found -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/
	kubectl delete --ignore-not-found -f ./fluentd-es-daemonset.yaml

influxdb-monitoring:
	# deploy all the RCs and SVCs in the add-on directory
	# WARNING : The heapster rc yaml has salt scripting that must be removed.
	kubectl create -f $(K8S_SRC)/cluster/addons/cluster-monitoring/influxdb/
	@echo "Grafana : http://localhost:$(PROXY_PORT)/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana/"

delete-influxdb-monitoring:
	kubectl delete --ignore-not-found -f $(K8S_SRC)/cluster/addons/cluster-monitoring/influxdb/
