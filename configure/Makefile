##  tested with make 3.82 on AL2

ANSIBLE := /usr/bin/ansible-playbook
KUBECTL := /usr/bin/kubectl
CURL := /usr/bin/curl
GIT := /usr/bin/git
HELM := /usr/bin/helm

ANSIBLE_FLAGS := -v
CURL_FLAGS := -LO

WORKING_DIR := $(HOME)/kubernetes
SRC_DIR := $(HOME)/src

CALICO_YML := $(WORKING_DIR)/calico/calico.yaml
DNS_YML := $(WORKING_DIR)/dns/coredns.yaml
TRAEFIK_YML := $(WORKING_DIR)/traefik/traefik-ds.yaml
TRAEFIK_RBAC_YML := $(WORKING_DIR)/traefik/traefik-rbac.yaml

KUBEFEST_DIR := $(SRC_DIR)/kubefest

PROMETHEUS_YML_DIR := $(SRC_DIR)/kube-prometheus/manifests
VERSION_KUBE_PROMETHEUS := 3c10936 # master

check:
	$(ANSIBLE) --version | head -1
	$(KUBECTL) version --short --client
	$(CURL) --version | head -1
	$(GIT) --version
	$(HELM) version --client

clean:
	rm -f $(WORKING_DIR)/*

cluster:  master node calico helm dns traefik

master:
	$(ANSIBLE) $(ANSIBLE_FLAGS) master.yml

node:
	$(ANSIBLE) $(ANSIBLE_FLAGS) node.yml

kube-system-summary:
	$(KUBECTL) get nodes -o wide
	$(KUBECTL) -n kube-system get all -o wide

calico:
	mkdir -p $(WORKING_DIR)/calico && cd $(WORKING_DIR)/calico && $(CURL) $(CURL_FLAGS) https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
	# replacement value needs to be in sync with "cluster_cidr" in vars.yml
	sed -i.ORIG 's_192.168.0.0/16_172.17.0.0/16_g' $(CALICO_YML)
	$(KUBECTL) apply -f $(CALICO_YML)
	$(KUBECTL) -n kube-system wait nodes --for=condition=Ready --timeout=60s -l cornet-role=node
	$(KUBECTL) -n kube-system wait pods  --for=condition=Ready --timeout=60s -l k8s-app=calico-node

helm:
	$(HELM) init
	$(KUBECTL) -n kube-system wait deploy --for=condition=Available --timeout=30s -l app=helm,name=tiller
	$(KUBECTL) -n kube-system wait pods --for=condition=Ready --timeout=30s -l app=helm,name=tiller

dns:
	mkdir -p $(WORKING_DIR)/dns && cd $(WORKING_DIR)/dns && $(CURL) $(CURL_FLAGS) https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.13/cluster/addons/dns/coredns/coredns.yaml.sed
	cp $(DNS_YML).sed $(DNS_YML)
	# replacement values needs to be in sync with "KUBELET_ARGS" in node.yml
	sed -i 's/$$DNS_SERVER_IP/172.27.0.53/g' $(DNS_YML)
	sed -i 's/$$DNS_DOMAIN/cluster.local/g' $(DNS_YML)
	# use the latest 1.2.x image version
	sed -i 's/coredns:1.2.6/coredns:1.3.1/g' $(DNS_YML)
	$(KUBECTL) apply -f $(DNS_YML)
	$(KUBECTL) -n kube-system wait deploy --for=condition=Available --timeout=30s -l k8s-app=kube-dns
	$(KUBECTL) -n kube-system wait pods --for=condition=Ready --timeout=30s -l k8s-app=kube-dns

traefik:
	mkdir -p $(WORKING_DIR)/traefik && cd $(WORKING_DIR)/traefik && $(CURL) $(CURL_FLAGS) "https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-{rbac,ds}.yaml"
	$(KUBECTL) apply -f $(TRAEFIK_RBAC_YML)
	# use a tag'ed image, instead of latest
	sed -i.ORIG 's/image: traefik$$/image: traefik:v1.7/g' $(TRAEFIK_YML)
	# enable access logging
	sed -i '/--api/p' $(TRAEFIK_YML)
	sed -i '0,/--api/s//--accesslog/' $(TRAEFIK_YML)
	$(KUBECTL) apply -f $(TRAEFIK_YML)
	# apply manifests from kubefest
	if [ -d $(KUBEFEST_DIR)/traefik ] ; then kubectl apply -f $(KUBEFEST_DIR)/traefik/ ; fi
	sleep 5 # kubectl can't wait on daemonset, so sleep until pods are scheduled
	$(KUBECTL) -n kube-system wait pods --for=condition=Ready --timeout=30s -l k8s-app=traefik-ingress-lb

monitoring:
	# setup kube-prometheus for monitoring
	mkdir -p $(SRC_DIR) && cd $(SRC_DIR) && $(GIT) clone -q https://github.com/coreos/kube-prometheus.git && cd $(SRC_DIR)/kube-prometheus && $(GIT) checkout -q $(VERSION_KUBE_PROMETHEUS)
	# prometheus-adapter requires extension API Server, so skip that
	rm -f $(PROMETHEUS_YML_DIR)/prometheus-adapter-*
	# copy customized manifests from kubefest
	if [ -d $(KUBEFEST_DIR)/kube-prometheus ] ; then cp $(KUBEFEST_DIR)/kube-prometheus/*.yaml $(PROMETHEUS_YML_DIR) ; fi
	$(KUBECTL) apply -f $(PROMETHEUS_YML_DIR)/
	$(KUBECTL) -n monitoring wait deploy --for=condition=Available --timeout=30s -l k8s-app=prometheus-operator
	$(KUBECTL) -n monitoring wait pods --for=condition=Ready --timeout=30s -l k8s-app=prometheus-operator
	# setup loki for logs
	helm repo add loki https://grafana.github.io/loki/charts
	# set helm options
	$(HELM) install loki/loki-stack --name loki --namespace monitoring --set promtail.entryParser=raw
	$(KUBECTL) -n monitoring wait deploy --for=condition=Available --timeout=90s -l app=loki
	$(KUBECTL) -n monitoring wait pods --for=condition=Ready --timeout=90s -l app=loki


.PHONY: check clean master node
.PHONY: cluster kube-system-summary
.PHONY: calico helm dns traefik
.PHONY: monitoring
