##  tested with make 3.82 on AL2

ANSIBLE := /usr/bin/ansible-playbook
KUBECTL := /usr/bin/kubectl
CURL := /usr/bin/curl
GIT := /usr/bin/git

ANSIBLE_FLAGS := -v
KUBE_FLAGS := -s http://master:8080
CURL_FLAGS := -LO

WORKING_DIR := $(HOME)/kubernetes
SRC_DIR := $(HOME)/src

FLANNEL_YML := $(WORKING_DIR)/flannel/kube-flannel.yml
CALICO_YML := $(WORKING_DIR)/calico/calico.yaml
DNS_YML := $(WORKING_DIR)/dns/coredns.yaml
TRAEFIK_YML := $(WORKING_DIR)/traefik/traefik-ds.yaml
TRAEFIK_RBAC_YML := $(WORKING_DIR)/traefik/traefik-rbac.yaml

PROMETHEUS_YML_DIR := $(SRC_DIR)/prometheus-operator/contrib/kube-prometheus/manifests
GRAFANA_YML := $(PROMETHEUS_YML_DIR)/grafana-deployment.yaml

VERSION_PROMETHEUS_OPERATOR := e381c49 # v0.29.0+

check:
	$(ANSIBLE) --version | head -1
	$(KUBECTL) version --short --client
	$(CURL) --version | head -1
	$(GIT) --version

clean:
	rm -f $(WORKING_DIR)/*

cluster:  master node calico

master:
	$(ANSIBLE) $(ANSIBLE_FLAGS) master.yml

node:
	$(ANSIBLE) $(ANSIBLE_FLAGS) node.yml

kube-system-summary:
	$(KUBECTL) $(KUBE_FLAGS) get nodes -o wide
	$(KUBECTL) $(KUBE_FLAGS) -n kube-system get all -o wide

# either flannel or calico can be used - calico is recommended and the default
flannel:
	mkdir -p $(WORKING_DIR)/flannel && cd $(WORKING_DIR)/flannel && $(CURL) $(CURL_FLAGS) https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
	# replacement value needs to be in sync with "cluster_cidr" in vars.yml
	sed -i.ORIG 's_10.244.0.0/16_172.17.0.0/16_g' $(FLANNEL_YML)
	$(KUBECTL) $(KUBE_FLAGS) apply -f $(FLANNEL_YML)

# either flannel or calico can be used - calico is recommended and the default
calico:
	mkdir -p $(WORKING_DIR)/calico && cd $(WORKING_DIR)/calico && $(CURL) $(CURL_FLAGS) https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
	# replacement value needs to be in sync with "cluster_cidr" in vars.yml
	sed -i.ORIG 's_192.168.0.0/16_172.17.0.0/16_g' $(CALICO_YML)
	$(KUBECTL) $(KUBE_FLAGS) apply -f $(CALICO_YML)

dns:
	mkdir -p $(WORKING_DIR)/dns && cd $(WORKING_DIR)/dns && $(CURL) $(CURL_FLAGS) https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.12/cluster/addons/dns/coredns/coredns.yaml.sed
	cp $(DNS_YML).sed $(DNS_YML)
	# replacement values needs to be in sync with "KUBELET_ARGS" in node.yml
	sed -i 's/$$DNS_SERVER_IP/172.27.0.53/g' $(DNS_YML)
	sed -i 's/$$DNS_DOMAIN/cluster.local/g' $(DNS_YML)
	$(KUBECTL) $(KUBE_FLAGS) apply -f $(DNS_YML)

traefik:
	mkdir -p $(WORKING_DIR)/traefik && cd $(WORKING_DIR)/traefik && $(CURL) $(CURL_FLAGS) "https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-{rbac,ds}.yaml"
	$(KUBECTL) $(KUBE_FLAGS) apply -f $(TRAEFIK_RBAC_YML)
	# use a tag'ed image, instead of latest
	sed -i.ORIG 's/image: traefik$$/image: traefik:v1.7/g' $(TRAEFIK_YML)
	# enable access logging
	sed -i '/--api/p' $(TRAEFIK_YML)
	sed -i '0,/--api/s//--accesslog/' $(TRAEFIK_YML)
	$(KUBECTL) $(KUBE_FLAGS) apply -f $(TRAEFIK_YML)

prometheus:
	mkdir -p $(SRC_DIR) && cd $(SRC_DIR) && $(GIT) clone -q https://github.com/coreos/prometheus-operator.git && cd $(SRC_DIR)/prometheus-operator && $(GIT) checkout -q $(VERSION_PROMETHEUS_OPERATOR)
	# prometheus-adapter requires extension API Server, so skip that
	rm -f $(PROMETHEUS_YML_DIR)/prometheus-adapter-*
	# allow anonymous auth in grafana
	sed -i '/volumeMount/i \
        env:\
          - name: GF_AUTH_ANONYMOUS_ENABLED\
            value: "true"' $(GRAFANA_YML)
	# upgrade grafana version to the latest
	sed -i 's/6.0.0-beta1/6.0.0-beta3/g' $(GRAFANA_YML)
	$(KUBECTL) $(KUBE_FLAGS) apply -f $(PROMETHEUS_YML_DIR)/

.PHONY: check clean master node cluster
.PHONY: kube-system-summary flannel calico
.PHONY: dns traefik prometheus
