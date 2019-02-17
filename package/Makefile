
# builds/packages kubernetes and dependent packages
# tested with GNU Make 3.82 on AL2

GOROOT := /usr/local/go

GIT := /usr/bin/git
CURL := /usr/bin/curl
YUM := sudo /usr/bin/yum
GO := $(GOROOT)/bin/go

CURL_FLAGS := -LO

VERSION_GOLANG := 1.11.5
VERSION_HAPROXY := 1.9.4

VERSION_KUBERNETES := v1.12.5
VERSION_CRIO := v1.12.5
VERSION_REGISTRY := v2.7.1
VERSION_ETCD := v3.3.12
VERSION_CNI := v0.7.1

VERSION_RUNC := 6635b4f # v1.0.0-rc6+dev
VERSION_BUILDAH := v1.6
VERSION_SKOPEO := v0.1.32
VERSION_LIBPOD := v1.0.0
VERSION_ATOMIC := v$(shell date +%Y%m%d)

export GOPATH := $(HOME)/go
export PATH := $(GOPATH)/bin:$(GOROOT)/bin:$(PATH)

help:
	@sed -ne '/sed/!s/## //p' $(MAKEFILE_LIST)

check:
	$(GIT) --version
	$(CURL) --version | head -1
	$(YUM) --version | head -1
	$(GO) version

clean:
	rm -rf $(GOPATH)

prep:
	mkdir -p $(GOPATH)/bin $(GOPATH)/dist $(GOPATH)/src $(GOPATH)/tmp


# one time setup
install-dependencies:  ## one time setup to install all dependencies to build
install-dependencies:  install-pkgs install-golang

install-pkgs:
	$(YUM) install -y git tree gcc-c++ system-rpm-config \
		glibc-devel glibc-static btrfs-progs-devel device-mapper-devel \
		glib2-devel gpgme-devel libassuan-devel libseccomp-devel libgpg-error-devel \
		pkgconfig systemd-devel

install-golang:
	cd /tmp && $(CURL) $(CURL_FLAGS) https://dl.google.com/go/go$(VERSION_GOLANG).linux-amd64.tar.gz
	sudo tar -C /usr/local -xzf /tmp/go$(VERSION_GOLANG).linux-amd64.tar.gz


# haproxy
haproxy:  ## build haproxy
haproxy:  clean prep prep-haproxy build-haproxy dist-haproxy

prep-haproxy:
	mkdir -p $(GOPATH)/tmp/haproxy

build-haproxy:
	cd $(GOPATH)/src && $(CURL) $(CURL_FLAGS) http://www.haproxy.org/download/1.9/src/haproxy-$(VERSION_HAPROXY).tar.gz
	cd $(GOPATH)/src && rm -rf haproxy-$(VERSION_HAPROXY) && tar xfz haproxy-$(VERSION_HAPROXY).tar.gz
	cd $(GOPATH)/src/haproxy-$(VERSION_HAPROXY) && make TARGET=linux2628 USE_SYSTEMD=1
	cd $(GOPATH)/src/haproxy-$(VERSION_HAPROXY)/contrib/systemd && make PREFIX=/usr
	cp $(GOPATH)/src/haproxy-$(VERSION_HAPROXY)/haproxy $(GOPATH)/tmp/haproxy
	cp $(GOPATH)/src/haproxy-$(VERSION_HAPROXY)/contrib/systemd/haproxy.service $(GOPATH)/tmp/haproxy

dist-haproxy:
	cd $(GOPATH)/tmp && tar cfz haproxy-$(VERSION_HAPROXY)-linux-amd64.tar.gz haproxy
	cp $(GOPATH)/tmp/haproxy-$(VERSION_HAPROXY)-linux-amd64.tar.gz $(GOPATH)/dist/


# kubernetes
kubernetes:  ## bundle kubernetes
kubernetes:  clean prep kubernetes-client kubernetes-node kubernetes-server

kubernetes-client:
	cd $(GOPATH)/tmp && $(CURL) $(CURL_FLAGS) https://dl.k8s.io/$(VERSION_KUBERNETES)/kubernetes-client-linux-amd64.tar.gz
	cp $(GOPATH)/tmp/kubernetes-client-linux-amd64.tar.gz $(GOPATH)/dist/kubernetes-client-$(VERSION_KUBERNETES)-linux-amd64.tar.gz

kubernetes-node:
	cd $(GOPATH)/tmp && $(CURL) $(CURL_FLAGS) https://dl.k8s.io/$(VERSION_KUBERNETES)/kubernetes-node-linux-amd64.tar.gz
	cd $(GOPATH)/tmp && rm -rf kubernetes && tar xfz kubernetes-node-linux-amd64.tar.gz
	cd $(GOPATH)/tmp/kubernetes && find -type f ! -name "kubectl" ! -name "kubelet" ! -name "kube-proxy" -delete
	cd $(GOPATH)/tmp && tar cfz kubernetes-node-$(VERSION_KUBERNETES)-linux-amd64.tar.gz kubernetes
	cp $(GOPATH)/tmp/kubernetes-node-$(VERSION_KUBERNETES)-linux-amd64.tar.gz $(GOPATH)/dist/

kubernetes-server:
	cd $(GOPATH)/tmp && $(CURL) $(CURL_FLAGS) https://dl.k8s.io/$(VERSION_KUBERNETES)/kubernetes-server-linux-amd64.tar.gz
	cd $(GOPATH)/tmp && rm -rf kubernetes && tar xfz kubernetes-server-linux-amd64.tar.gz
	cd $(GOPATH)/tmp/kubernetes && find -type f ! -name "kubectl" ! -name "kube-apiserver" ! -name "kube-controller-manager" ! -name "kube-scheduler" -delete
	cd $(GOPATH)/tmp && tar cfz kubernetes-server-$(VERSION_KUBERNETES)-linux-amd64.tar.gz kubernetes
	cp $(GOPATH)/tmp/kubernetes-server-$(VERSION_KUBERNETES)-linux-amd64.tar.gz $(GOPATH)/dist/


# atomic
atomic:  ## build atomic
atomic:  clean prep prep-atomic runc buildah skopeo libpod dist-atomic

prep-atomic:
	mkdir -p $(GOPATH)/src/github.com/opencontainers
	mkdir -p $(GOPATH)/src/github.com/containers
	mkdir -p $(GOPATH)/tmp/atomic

runc:
	cd $(GOPATH)/src/github.com/opencontainers && $(GIT) clone -q https://github.com/opencontainers/runc && cd runc && $(GIT) checkout -q $(VERSION_RUNC) && make runc
	mkdir -p $(GOPATH)/tmp/atomic/runc &&  cp $(GOPATH)/src/github.com/opencontainers/runc/runc  $(GOPATH)/tmp/atomic/runc/

buildah:
	cd $(GOPATH)/src/github.com/containers && $(GIT) clone -q https://github.com/containers/buildah && cd buildah && $(GIT) checkout -q $(VERSION_BUILDAH) && make buildah
	mkdir -p $(GOPATH)/tmp/atomic/buildah && cp $(GOPATH)/src/github.com/containers/buildah/buildah  $(GOPATH)/tmp/atomic/buildah

skopeo:
	cd $(GOPATH)/src/github.com/containers && $(GIT) clone -q https://github.com/containers/skopeo && cd skopeo && $(GIT) checkout -q $(VERSION_SKOPEO) && make binary-local BUILDTAGS=containers_image_ostree_stub
	mkdir -p $(GOPATH)/tmp/atomic/skopeo && cp $(GOPATH)/src/github.com/containers/skopeo/skopeo  $(GOPATH)/tmp/atomic/skopeo

libpod:
	cd $(GOPATH)/src/github.com/containers && $(GIT) clone -q https://github.com/containers/libpod && cd libpod && $(GIT) checkout -q $(VERSION_LIBPOD) && make install.tools && make binaries
	mkdir -p $(GOPATH)/tmp/atomic/libpod && cp $(GOPATH)/src/github.com/containers/libpod/bin/podman  $(GOPATH)/tmp/atomic/libpod

dist-atomic:
	cd $(GOPATH)/tmp && tar cfz atomic-$(VERSION_ATOMIC)-linux-amd64.tar.gz atomic
	cp $(GOPATH)/tmp/atomic-$(VERSION_ATOMIC)-linux-amd64.tar.gz $(GOPATH)/dist/


# crio
crio:  ## build crio
crio:  clean prep prep-crio build-crio dist-crio

prep-crio:
	mkdir -p $(GOPATH)/src/github.com/kubernetes-sigs
	mkdir -p $(GOPATH)/tmp/cri-o

build-crio:
	cd $(GOPATH)/src/github.com/kubernetes-sigs && $(GIT) clone -q https://github.com/kubernetes-sigs/cri-o && cd cri-o && $(GIT) checkout -q $(VERSION_CRIO) && make binaries
	cp -R $(GOPATH)/src/github.com/kubernetes-sigs/cri-o/bin $(GOPATH)/tmp/cri-o
	cd $(GOPATH)/tmp/cri-o && ./bin/crio --config="" config  >  $(GOPATH)/tmp/cri-o/crio.conf
	cp $(GOPATH)/src/github.com/kubernetes-sigs/cri-o/seccomp.json $(GOPATH)/tmp/cri-o

dist-crio:
	cd $(GOPATH)/tmp && tar cfz cri-o-$(VERSION_CRIO)-linux-amd64.tar.gz cri-o
	cp $(GOPATH)/tmp/cri-o-$(VERSION_CRIO)-linux-amd64.tar.gz $(GOPATH)/dist/


# registry / docker-distribution
registry:  ## build registry
registry:  clean prep prep-registry build-registry dist-registry

prep-registry:
	mkdir -p $(GOPATH)/src/github.com/docker
	mkdir -p $(GOPATH)/tmp/docker-distribution

build-registry:
	cd $(GOPATH)/src/github.com/docker && $(GIT) clone -q https://github.com/docker/distribution && cd distribution && $(GIT) checkout -q $(VERSION_REGISTRY) && make binaries
	cp $(GOPATH)/src/github.com/docker/distribution/bin/registry $(GOPATH)/tmp/docker-distribution

dist-registry:
	cd $(GOPATH)/tmp && tar cfz docker-distribution-$(VERSION_REGISTRY)-linux-amd64.tar.gz docker-distribution
	cp $(GOPATH)/tmp/docker-distribution-$(VERSION_REGISTRY)-linux-amd64.tar.gz $(GOPATH)/dist/


# etcd
etcd:  ## bundle etcd
etcd:  clean prep
	cd $(GOPATH)/dist && $(CURL) $(CURL_FLAGS) https://github.com/etcd-io/etcd/releases/download/$(VERSION_ETCD)/etcd-$(VERSION_ETCD)-linux-amd64.tar.gz

# cni
cni:  ## bundle cni
cni:  clean prep
	cd $(GOPATH)/dist && $(CURL) $(CURL_FLAGS) https://github.com/containernetworking/plugins/releases/download/$(VERSION_CNI)/cni-plugins-amd64-$(VERSION_CNI).tgz


# phony
.PHONY: help check clean prep
.PHONY: install-dependencies install-pkgs install-golang
.PHONY: haproxy prep-haproxy build-haproxy dist-haproxy
.PHONY: kubernetes kubernetes-client kubernetes-node kubernetes-server
.PHONY: atomic prep-atomic runc buildah skopeo libpod dist-atomic
.PHONY: crio prep-crio build-crio dist-crio
.PHONY: registry prep-registry build-registry dist-registry
.PHONY: etcd cni