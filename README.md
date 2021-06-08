# CentOS-EKS

## Overview

To build an EKS-support Ubuntu AMI for Graviton2, you need to install EKS runtimes and scripts. Since that is a complex process, AWS provides 'Packer' scripts and configurations to create custom AMIs for use with Amazon EKS via self-managed Auto Scaling Groups and Managed Node Groups. Packer is a tool to bake a custom AMI in an automatic way. More information about Packer can be found [here](https://learn.hashicorp.com/tutorials/packer/get-started-install-cli). The Packet scripts from AWS are found at [this link](https://github.com/aws-samples/amazon-eks-custom-amis)

## Changes to AWS Packer scripts for Graviton2

AWS Packer scrips support Intel-based instances by default. For Graviton2 or ARM64 architecture, the following changes should be applied to the original AWS Packer scripts in order to build your AMI for Graviton2:

## Step 1: Update VPC and subnet IDs in Makefile

Change the VPC_ID, SUBNET_ID, and AWS_REGION properly. Note that the subnet for a packer instance must support automatic public IP assignment. Please check if the automatic public IP assignment is ON.
Here is an example of the head of Makefile

```
VPC_ID := vpc-0eccdebe81b47954f
SUBNET_ID := subnet-048d6fdc454a3c3c5
AWS_REGION := us-east-1
The packer configuration files (*.json) support the x86 processor. For Graviton, you need to change the arch and ami variables:
Edit “amazon-eks-node-ubuntu2004.json” as follows:
22,23c22,23
<     "source_ami_arch":"x86_64",
<     "source_ami_name":"ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*",
---
>     "source_ami_arch":"arm64",
>     "source_ami_name":"ubuntu/images/hvm-ssd/ubuntu-focal-20.04-arm64-server-*",
44c44
<       "instance_type":"m5.xlarge",
---
>       "instance_type":"c6g.large",
```

## Step 2: Change a script to install ARM-based 'jq'

The packer script puts ‘jq’ for json data but it installs Intel-based jq which doesn’t work when ‘/etc/eks/ bootstrap.sh’ runs. The bootstrap fails and can’t register this instance to a cluster.
 
To fix this, you can change the code lines in ‘files/functions.sh’ as follows:
```
vi files/functions.sh
---
"files/functions.sh" 485 lines --54%--
install_jq() {
    #curl -sL -o /usr/bin/jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
    #chmod +x /usr/bin/jq
    apt-get install -y jq
}
---
```

## Step 3: Bake your AMI

Finally, you are ready to bake your Ubuntu AMI for Ubuntu 20.04 and EKS 1.19:
```bash
make build-ubuntu2004-1.19
```

The new AMI can be used to launch a Graviton2 instance (i.e., instance type c6g or c6gd) and a worker node to attach a EKS cluster. Packer shows the AMI ID of the baked AMI, which can be also found at AWS management console.

