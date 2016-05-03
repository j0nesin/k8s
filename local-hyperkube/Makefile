# Run kubernetes locally using containers

.PHONY:	run

K8S_VERSION = v1.2.2

DNS_REPLICAS=1
DNS_DOMAIN=cluster.local
DNS_SERVER_IP=10.0.0.10

default: build

get-stable-version-num:
	curl -sS https://storage.googleapis.com/kubernetes-release/release/stable.txt

get-kubectl:
	wget http://storage.googleapis.com/kubernetes-release/release/$(K8S_VERSION)/bin/linux/amd64/kubectl
	chmod 755 kubectl

config-cluster:
	kubectl config set-cluster hyperkube --server=http://localhost:8080
	kubectl config set-context hyperkube --cluster=hyperkube
	kubectl config use-context hyperkube

run:
	docker run \
    --volume=/:/rootfs:ro \
    --volume=/sys:/sys:ro \
    --volume=/var/lib/docker/:/var/lib/docker:rw \
    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
    --volume=/var/run:/var/run:rw \
    --net=host \
    --pid=host \
    --privileged=true \
    --name=kubelet \
    -d \
    gcr.io/google_containers/hyperkube-amd64:$(K8S_VERSION) \
    /hyperkube kubelet \
        --containerized \
        --hostname-override="127.0.0.1" \
        --address="0.0.0.0" \
        --api-servers=http://localhost:8080 \
        --config=/etc/kubernetes/manifests \
        --cluster-dns=$(DNS_SERVER_IP) \
        --cluster-domain=$(DNS_DOMAIN) \
        --allow-privileged=true --v=2

stop:
	-docker stop kubelet `docker ps --filter "name=k8s" -q`
	-docker rm -v kubelet `docker ps --filter "name=k8s" -aq`

K8S_SRC = ./kubernetes
# get kubernetes repo with add-on configs
get-src:
	git clone --depth 1 https://github.com/kubernetes/kubernetes.git
# update kubernetes src
update-src:
	cd $(PWD)/kubernetes && git pull

kube-system-namespace:
	-kubectl create -f $(K8S_SRC)/docs/getting-started-guides/docker-multinode/kube-system.yaml

# ERROR : Does not seem to work when deployed.
dns-add-on: kube-system-namespace
	sed -e "s/{{ pillar\['dns_replicas'\] }}/${DNS_REPLICAS}/g;s/{{ pillar\['dns_domain'\] }}/${DNS_DOMAIN}/g;s/{{ pillar\['dns_server'\] }}/${DNS_SERVER_IP}/g" $(K8S_SRC)/cluster/addons/dns/skydns-rc.yaml.in > ./skydns-rc.yaml
	kubectl create -f skydns-rc.yaml
	sed -e "s/{{ pillar\['dns_replicas'\] }}/${DNS_REPLICAS}/g;s/{{ pillar\['dns_domain'\] }}/${DNS_DOMAIN}/g;s/{{ pillar\['dns_server'\] }}/${DNS_SERVER_IP}/g" $(K8S_SRC)/cluster/addons/dns/skydns-svc.yaml.in > ./skydns-svc.yaml
	kubectl create -f skydns-svc.yaml

dashboard-add-on:
