# Package
The Makefile builds/bundles the packages and uploads them to a s3 bucket.

## Overview
The below table has information on how packages are build/bundled. See `Makefile` for details.

| Package | Tier | Notes |
| --- | --- | --- |
| atomic | bastion,node | build from sources |
| cri-o | node | build from sources |
| docker-distribution | master | build from sources |
| etcd | master | bundle from a binary release |
| haproxy | bastion | build from sources |
| helm | bastion | bundle from a binary release |
| kubernetes-client | bastion | bundle from a binary release |
| kubernetes-node | node | bundle from a binary release |
| kubernetes-server | master | bundle from a binary release |

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
