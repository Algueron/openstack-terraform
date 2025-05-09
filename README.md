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

## SSH Key

- Create a keypair
````bash
openstack keypair create --private-key terraform.key --type ssh terraform-key
````

- Set the correct permissions on the private key
````bash
chmod 400 terraform.key
````

## Virtual Machine

- Create an Ubuntu virtual machine
````bash
openstack server create --flavor t2.small --image ubuntu-server-24.04 --network infrastructure-net --security-group default --security-group allow-ssh --key-name terraform-key terraform
````

- Create a Floating IP
````bash
openstack floating ip create public-net
````

- Store the IP generated
````bash
TF_FLOATING_IP=$(openstack floating ip list -f value -c "Floating IP Address")
````

- Assign the floating IP to the Virtual Machine
````bash
openstack server add floating ip terraform $TF_FLOATING_IP
````

## Terraform Setup

- Log into the Terraform VM using the floating ip and private key
````bash
ssh -i terraform.key ubuntu@$TF_FLOATING_IP
````

- Install gpg.
````bash
sudo apt install -y gpg
````

- Download the signing key to a new keyring.
````bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
````

- Verify the key's fingerprint.
````bash
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint
````

- Add the official HashiCorp Linux repository.
````bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
````

- Update and install Terraform.
````bash
sudo apt-get update && sudo apt-get install terraform
````
