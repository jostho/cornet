# Package
The Makefile builds/bundles requires packages and uploads them to a s3 bucket.

## Setup

Provision a bastion node - which can be used as a build machine

    ansible-playbook -v provision/bastion.yml

Copy Makefile to this node

    scp package/Makefile ec2-user@54.x.x.x:~/

Ssh into bastion

    ssh -A ec2-user@54.x.x.x

Then configure the bastion to build

    make install-dependencies && make check

Teardown bastion after use

    ansible-playbook -v teardown/ec2.yml

## Build

Ssh into bastion. Run make to see the help

    make

Bundle kubernetes

    make kubernetes

Verify the artifacts

    ls ~/go/dist

Sync the artifacts to the s3 bucket

    cd ~/go/dist/ && aws s3 sync . s3://cornet/archives/kubernetes/
