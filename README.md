# Cornet
This sets up a working kubernetes cluster on Amazon Linux 2 in AWS.

This was tested with Fedora on localhost.

## Cluster environment
* etcd 3.3
* docker-distribution 2.7
* kubernetes 1.12
* cri-o 1.12
* calico 3.5

## Local environment
* fedora 29
* ansible 2.7
* python3-boto
* python3-boto3

## Prerequisites
1. Access to EC2 service
2. A VPC with public and private subnets

Update `vpc_id` and `subnet_id` values in `vars_aws.yml` to use an existing vpc

Or create a new vpc

    ansible-playbook -v provision/vpc.yml

## Setup

Create a `~/.ansible.cfg` config file, with the below contents

    [defaults]
    host_key_checking = False
    retry_files_enabled = False
    callback_whitelist = profile_tasks

Import your keypair using aws console or the cli

Create IAM role (if not created already)

    ansible-playbook -v provision/role.yml

Create ec2 security groups for the bastion and kubernetes cluster (if not created already)

    ansible-playbook -v provision/sg.yml

Authorize your current IP with bastion SG (if not authorized already)

    ansible-playbook -v configure/sg.yml

See [package README](package/README.md) to find out how to build artifacts and push to a s3 bucket

## Provison the cluster

Provision a bastion node

    ansible-playbook -v provision/bastion.yml

Provision the kubernetes cluster

    ansible-playbook -v provision/master.yml
    ansible-playbook -v provision/node.yml

Configure bastion node from localhost

    ansible-playbook -v -i ./hosts configure/bastion.yml

Ssh into bastion

    ssh -A ec2-user@54.x.x.x

Then configure the cluster from bastion

    cd ~/src/cornet/configure && make cluster

Teardown ec2 instances after use

    ansible-playbook -v teardown/ec2.yml

Teardown ec2 security groups after use

    ansible-playbook -v teardown/sg.yml

## Build a container image

Ssh into bastion. Build [ecgo](https://github.com/jostho/ecgo) container image

    mkdir -p ~/go/src && cd ~/go/src && git clone https://github.com/jostho/ecgo.git && cd ecgo && make image

Verify the built image

    buildah images
    buildah inspect jostho/ecgo:0.1.0

Push the image to the private registry

    buildah push --tls-verify=false jostho/ecgo:0.1.0 docker://registry:5000/ecgo:0.1.0

## Deploy the image

Ssh into bastion. Verify the image in the registry

    skopeo inspect --tls-verify=false docker://registry:5000/ecgo:0.1.0

Create a pod

    kubectl run ecgo --image=registry:5000/ecgo:0.1.0 --port=8000 --restart=Never

Verify the pod

    kubectl get pods -o wide
    kubectl logs ecgo
