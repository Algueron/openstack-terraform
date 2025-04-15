# Terraform on Openstack
This project intends to deploy a Terraform virtual machine on an Openstack cloud deployed using Kolla (see [this repository](https://github.com/Algueron/openstack-home)).
The instructions must be executed on the deployment node.

## Login

- Switch to Virtual env
````bash
source ~/kolla/venv/bin/activate
````

- Log as Openstack admin user
````bash
. /etc/kolla/admin-openrc.sh
````

## Project and user creation

- Create the infra project
````bash
openstack project create --description 'Terraform Hosts for provisioning' infrastructure --domain default
````

- Generate a random password
````bash
export TF_PASSWORD=$(openssl rand -base64 18)
````

- Create a Terraform user
````bash
openstack user create --project infrastructure --password $TF_PASSWORD terraform
````

- Assign the role member to terraform
````bash
openstack role add --user terraform --project infrastructure member
````

- Download the terraform [credentials file](terraform-openrc.sh)
````bash
wget https://raw.githubusercontent.com/Algueron/openstack-terraform/main/terraform-openrc.sh
````

- Fill the Terraform password
````bash
sed -i -e "s~TF_PASSWORD~$TF_PASSWORD~g" terraform-openrc.sh
````

## Network setup

- Log as Openstack terraform user
````bash
source terraform-openrc.sh
````

- Create a private network for infrastructure
````bash
openstack network create --enable infrastructure-net
````

- Create a private subnet
````bash
openstack subnet create --subnet-range "192.168.75.0/24" --dhcp --ip-version 4 --dns-nameserver "192.168.1.15" --network infrastructure-net infrastructure-subnet
````

- Create a router connected to the Provider network
````bash
openstack router create --enable --external-gateway public-net infrastructure-router
````

- Connect the router to the infrastructure network
````bash
openstack router add subnet infrastructure-router infrastructure-subnet
````

## Security groups

- Create a security group to allow SSH
````bash
openstack security group create --stateful allow-ssh
````

- Add the rule to allow SSH
````bash
openstack security group rule create --remote-ip "192.168.0.0/24" --protocol tcp --dst-port 22 --ingress allow-ssh
````

