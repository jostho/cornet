# Cornet
This sets up a working kubernetes cluster on Amazon Linux 2 in AWS.

This was tested with Fedora on localhost.

## Cluster environment
* [etcd](https://github.com/etcd-io/etcd) 3.3
* [docker-distribution](https://github.com/docker/distribution) 2.7
* [kubernetes](https://github.com/kubernetes/kubernetes) 1.13
* [cri-o](https://github.com/kubernetes-sigs/cri-o) 1.13
* [calico](https://github.com/projectcalico/calico) 3.6
* [helm](https://github.com/helm/helm) 2.13
* [coredns](https://github.com/coredns/coredns) 1.3

## AWS resources
* VPC with subnets
* EC2 instances & security groups
* S3 bucket
* IAM roles

## Local environment
* fedora 29
* ansible 2.7
* python3-boto 2.45
* python3-boto3 1.7

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
    [ssh_connection]
    pipelining = True

Import your keypair using aws console or the cli. Update `key_name` in `vars_aws.yml` to use that keypair

Create IAM roles (if not created already)

    ansible-playbook -v provision/role.yml

Create ec2 security groups for the bastion and kubernetes cluster (if not created already)

    ansible-playbook -v provision/sg.yml

Authorize your current IP with bastion security group (if not authorized already)

    ansible-playbook -v configure/sg.yml

See [package README](package/README.md) to find out how to build artifacts and push to a s3 bucket

## Provision the cluster

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

Setup monitoring for the cluster. This installs [kube-prometheus](https://github.com/coreos/kube-prometheus) manifests

    cd ~/src/cornet/configure && make monitoring

Teardown ec2 instances after use

    ansible-playbook -v teardown/ec2.yml

Teardown ec2 security groups after use

    ansible-playbook -v teardown/sg.yml

## Build a container image

Ssh into bastion. Build [ecgo](https://github.com/jostho/ecgo) container image

    cd ~/src && git clone https://github.com/jostho/ecgo.git && cd ecgo && make image

Verify the built image

    buildah images
    buildah inspect jostho/ecgo:v0.2.0

Push the image to the private registry

    buildah push --tls-verify=false jostho/ecgo:v0.2.0 docker://registry:5000/jostho/ecgo:v0.2.0

## Deploy the image

Ssh into bastion. Verify the image in the registry

    skopeo inspect --tls-verify=false docker://registry:5000/jostho/ecgo:v0.2.0

Create a pod

    kubectl run ecgo --image=registry:5000/jostho/ecgo:v0.2.0 --port=8000 --restart=Never

Verify the pod

    kubectl get pods -o wide
    kubectl logs ecgo
