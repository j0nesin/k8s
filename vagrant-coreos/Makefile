# Steps for vagrant based multi-node cluster

.PHONY:	kubeconfig dashboard-add-on

K8S_SRC = ./kubernetes
COREOS_K8S_SRC = ./coreos-kubernetes

# get other repos needed to run all commands
get-src:
	git clone --depth 1 https://github.com/kubernetes/kubernetes.git
	git clone --depth 1 https://github.com/coreos/coreos-kubernetes.git
# update other repos
update-src:
	cd $(PWD)/kubernetes && git pull
	cd $(PWD)/coreos-kubernetes && git pull

run:
	cd $(COREOS_K8S_SRC)/multi-node/vagrant
	vagrant up

status:
	cd $(COREOS_K8S_SRC)/multi-node/vagrant
	vagrant status

stop:
	cd $(COREOS_K8S_SRC)/multi-node/vagrant
	vagrant halt c1 e1 w1 w2

delete: stop
	cd $(COREOS_K8S_SRC)/multi-node/vagrant
	vagrant destroy

kubeconfig:
	kubectl config set-cluster coreos-vagrant-multi --server=https://172.17.4.101:443 --certificate-authority=$(COREOS_K8S_SRC)/multi-node/vagrant/ssl/ca.pem
	kubectl config set-credentials coreos-vagrant-multi --certificate-authority=$(COREOS_K8S_SRC)/multi-node/vagrant/ssl/ca.pem --client-key=$(COREOS_K8S_SRC)/multi-node/vagrant/ssl/admin-key.pem --client-certificate=$(COREOS_K8S_SRC)/multi-node/vagrant/ssl/admin.pem
	kubectl config set-context coreos-vagrant-multi --cluster=coreos-vagrant-multi --user=coreos-vagrant-multi
	kubectl config use-context coreos-vagrant-multi

dashboard-add-on:
	kubectl create -f $(K8S_SRC)/cluster/addons/dashboard/dashboard-controller.yaml
	kubectl create -f $(K8S_SRC)/cluster/addons/dashboard/dashboard-service.yaml
	kubectl proxy --port 8011
	echo "Access dashboard at http://localhost:8011/ui"

delete-dashboard-add-on:
	kubectl delete -f $(K8S_SRC)/cluster/addons/dashboard/dashboard-controller.yaml
	kubectl delete -f $(K8S_SRC)/cluster/addons/dashboard/dashboard-service.yaml

elk-add-on:
	# CoreOS has a bug that prevents logging from working.  The following mods are needed:
	# See https://github.com/coreos/coreos-kubernetes/issues/322 for issue details
	# - Add the following line to the kubelet wrapper startup configuration at /etc/systemd/system/kubelet.service
	# - Environment="RKT_OPTS=--volume container,kind=host,source=/var/log/containers,readOnly=false --mount volume=container,target=/var/log/containers"
	# - mkdir /etc/log/containers
	# - chmod 755 /etc/log/containers
	kubectl create -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/es-controller.yaml
	kubectl create -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/es-service.yaml
	kubectl create -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/kibana-controller.yaml
	kubectl create -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/kibana-service.yaml
	# need fluentd-elasticsearch container to run on all nodes.  Two options for this:
	# - Copy $(K8S_SRC)/cluster/saltbase/salt/fluentd-es/fluentd-es.yaml to /etc/kubernetes/manifests on all machines
	# - Deploy container as daemon-set pod using custom file.  Keep custom file versions updated with K8S src file listed above
	kubectl create -f ./fluentd-es-daemonset.yaml

delete-elk-add-on:
	kubectl delete -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/es-controller.yaml
	kubectl delete -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/es-service.yaml
	kubectl delete -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/kibana-controller.yaml
	kubectl delete -f $(K8S_SRC)/cluster/addons/fluentd-elasticsearch/kibana-service.yaml
	kubectl delete -f ./fluentd-es-daemonset.yaml
