# Package
The Makefile builds/bundles the packages and uploads them to a s3 bucket.

## Overview
The below table has information on how packages are build/bundled. See `Makefile` for details.

| Package | Tier | Notes |
| --- | --- | --- |
| atomic <sup>1</sup>| bastion,node | build from sources |
| [cri-o](https://github.com/kubernetes-sigs/cri-o) | node | build from sources |
| [docker-distribution](https://github.com/docker/distribution) | master | build from sources |
| [etcd](https://github.com/etcd-io/etcd) | master | bundle from a binary release |
| [haproxy](https://www.haproxy.org/) | bastion | build from sources |
| [helm](https://github.com/helm/helm) | bastion | bundle from a binary release |
| [kubernetes-client](https://github.com/kubernetes/kubernetes) | bastion | bundle from a binary release |
| [kubernetes-node](https://github.com/kubernetes/kubernetes) | node | bundle from a binary release |
| [kubernetes-server](https://github.com/kubernetes/kubernetes) | master | bundle from a binary release |

[1] `atomic` archive contains [runc](https://github.com/opencontainers/runc), [buildah](https://github.com/containers/buildah), [skopeo](https://github.com/containers/skopeo) and [podman](https://github.com/containers/libpod).

## Setup

Provision a build node

    ansible-playbook -v provision/build.yml

Configure build node from localhost

    ansible-playbook -v -i ./hosts configure/build.yml

Ssh into build node

    ssh -A ec2-user@54.x.x.x

Teardown build after use

    ansible-playbook -v teardown/ec2.yml

## Build

Ssh into build. Install golang and check

    make install-golang && make check

Run make to see the help

    make

Build everything

    make dist

Verify the artifacts

    tree ~/dist

Sync the artifacts to the s3 bucket

    aws s3 sync ~/dist/ $CORNET_BUCKET_BASE_PATH
