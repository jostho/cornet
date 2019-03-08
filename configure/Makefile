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
PROMETHEUS_YML_DIR := $(SRC_DIR)/prometheus-operator/contrib/kube-prometheus/manifests
LOKI_HELM_DIR := $(SRC_DIR)/loki/production/helm

VERSION_PROMETHEUS_OPERATOR := 1f2cf36 # v0.29.0+
VERSION_LOKI := 067c5af # master

check:
	$(ANSIBLE) --version | head -1
	$(KUBECTL) version --short --client
	$(CURL) --version | head -1
	$(GIT) --version
	$(HELM) version --client

clean:
	rm -f $(WORKING_DIR)/*

cluster:  master node calico helm dns

master:
	$(ANSIBLE) $(ANSIBLE_FLAGS) master.yml

node:
	$(ANSIBLE) $(ANSIBLE_FLAGS) node.yml

kube-system-summary:
	$(KUBECTL) get nodes -o wide
	$(KUBECTL) -n kube-system get all -o wide

calico:
	mkdir -p $(WORKING_DIR)/calico && cd $(WORKING_DIR)/calico && $(CURL) $(CURL_FLAGS) https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
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
	mkdir -p $(WORKING_DIR)/dns && cd $(WORKING_DIR)/dns && $(CURL) $(CURL_FLAGS) https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.12/cluster/addons/dns/coredns/coredns.yaml.sed
	cp $(DNS_YML).sed $(DNS_YML)
	# replacement values needs to be in sync with "KUBELET_ARGS" in node.yml
	sed -i 's/$$DNS_SERVER_IP/172.27.0.53/g' $(DNS_YML)
	sed -i 's/$$DNS_DOMAIN/cluster.local/g' $(DNS_YML)
	# use the latest 1.2.x image version
	sed -i 's/coredns:1.2.2/coredns:1.2.6/g' $(DNS_YML)
	$(KUBECTL) apply -f $(DNS_YML)
	$(KUBECTL) -n kube-system wait deploy --for=condition=Available --timeout=30s -l k8s-app=kube-dns
	$(KUBECTL) -n kube-system wait pods --for=condition=Ready --timeout=30s -l k8s-app=kube-dns

traefik:
	mkdir -p $(WORKING_DIR)/traefik && cd $(WORKING_DIR)/traefik && $(CURL) $(CURL_FLAGS) "https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-{rbac,ds}.yaml"
	$(KUBECTL) apply -f $(TRAEFIK_RBAC_YML)
	# use a tag'ed image, instead of latest
	sed -i.ORIG 's/image: traefik$$/image: traefik:v1.7/g' $(TRAEFIK_YML)
	# enable access logging
	sed -i '/--api/p' $(TRAEFIK_YML)
	sed -i '0,/--api/s//--accesslog/' $(TRAEFIK_YML)
	$(KUBECTL) apply -f $(TRAEFIK_YML)

prometheus:
	mkdir -p $(SRC_DIR) && cd $(SRC_DIR) && $(GIT) clone -q https://github.com/coreos/prometheus-operator.git && cd $(SRC_DIR)/prometheus-operator && $(GIT) checkout -q $(VERSION_PROMETHEUS_OPERATOR)
	# prometheus-adapter requires extension API Server, so skip that
	rm -f $(PROMETHEUS_YML_DIR)/prometheus-adapter-*
	# copy customized manifests from kubefest
	if [ -d $(KUBEFEST_DIR)/prometheus-operator ] ; then cp $(KUBEFEST_DIR)/prometheus-operator/*.yaml $(PROMETHEUS_YML_DIR) ; fi
	$(KUBECTL) apply -f $(PROMETHEUS_YML_DIR)/

loki:
	mkdir -p $(SRC_DIR) && cd $(SRC_DIR) && $(GIT) clone -q https://github.com/grafana/loki.git && cd $(SRC_DIR)/loki && $(GIT) checkout -q $(VERSION_LOKI)
	# set promtail entry parser
	$(HELM) install $(LOKI_HELM_DIR) --name loki --namespace logging --set promtail.entryParser=raw

.PHONY: check clean master node
.PHONY: cluster kube-system-summary
.PHONY: calico helm dns
.PHONY: traefik prometheus loki
