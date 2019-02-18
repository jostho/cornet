# Package
The Makefile builds/bundles requires packages and uploads them to a s3 bucket.

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
