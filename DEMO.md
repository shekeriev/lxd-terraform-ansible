# LXD + Terraform + Ansible (Demo Steps)

## Setup the Environment

Create the two machines with

```bash
vagrant up 
```

## Manual Exploration of Linux Containers

Enter the LXD machine

```bash
vagrant ssh lxd
```

Check if there are any containers (there shouldn't be any)

```bash
lxc list
```

List the locally stored images (none so far)

```bash
lxc image ls
```

Check the registered remote image servers

```bash
lxc remote ls
```

Search for Debian images on one of them

```bash
lxc image ls images: debian
```

Start a Debian-based container

```bash
lxc launch images:debian/12/cloud mydebian
```

Once the container is started, ask again for the list of containers

```bash
lxc list
```

Okay, we can see it

Note down its IP address and use it in some of the commands that follow

Let's open a BASH session in the container

```bash
lxc exec mydebian -- bash
```

Now, we can execute regular commands like

```bash
hostname
```

```bash
uname -a
```

Let's get the list of running processes

```bash
ps ax
```

Check the status of the SSH service

```bash
systemctl status ssh
```

It appears that there is no SSH service running (and installed)

Let's correct this

Update the repositories

```bash
apt update
```

Install the missing package

```bash
apt install openssh-server
```

Check the status of the newly installed service

```bash
systemctl status ssh
```

Okay, it is active and running

We can see it in the list of running processes as well

```bash
ps ax
```

Exit from the container to the LXD host

```bash
exit
```

Now try to open an SSH session to the container

```bash
ssh <container-ip>
```

Ha, what is the password of the user in the container? Is it the same as on the VM? Is there such user in the container?

Let's try with the root user as we know there is one in the container

```bash
ssh root@10.167.209.50
```

Again, what is its password? No clue

Open again a BASH session

```bash
lxc exec mydebian -- bash
```

Add a regular user - vagrant in our case

```bash
useradd -m -s /bin/bash -c 'Vagrant User' vagrant
```

Set its password (we will use the same as on the VM - vagrant)

```bash
passwd vagrant
```

Let's make it a sudoer

```bash
echo 'vagrant ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/vagrant
```

And exit to the LXD host

```bash
exit
```

Try to open again an SSH session

```bash
ssh 10.167.209.50
```

It works

Try it with a few commands, including sude

```bash
pwd
```

```bash
sudo apt update
```

Everything is perfect

Exit to the LXD host

```bash
exit
```

Let's create an SSH key pair to use it for authentication instead of the password

```bash
ssh-keygen
```

Then copy the public key to the container

```bash
ssh-copy-id <container-ip>
```

And try the new authentication method

```bash
ssh <container-ip>
```

Close the session and exit to the LXD host

```bash
exit
```

Let's add a port forwarding rule to use instead of the container's IP address

```bash
lxc config device add mydebian ssh proxy listen=tcp:0.0.0.0:10101 connect=tcp:127.0.0.1:22
```

Check that the new port appears as listening

```bash
sudo ss -ntpl
```

Try to connect through it

```bash
ssh -p 10101 localhost
```

Let's install an NGINX server

```bash
sudo apt install curl nginx
```

And then change the default index page

```bash
echo 'It works (in Linux Container)!' | sudo tee /var/www/html/index.html
```

Finally, we can test locally from within the container

```bash
curl http://localhost
```

It works

Close the session and exit to the LXD host

```bash
exit
```

Now visit the page by using the container's IP address

```bash
curl http://<container-ip>
```

It works as well

Let's add another proxy device or port-forwarding rule, but this time for the NGINX service

```bash
lxc config device add mydebian http proxy listen=tcp:0.0.0.0:10102 connect=tcp:127.0.0.1:80
```

And test again

```bash
curl http://localhost:10102
```

This one works as well

We can delete the container with

```bash
lxc delete mydebian --force
```

And check that it is gone with

```bash
lxc list
```

Exit to the host

```bash
exit
```

## LXD + Terraform

Open a session to the workstation VM

```bash
vagrant ssh workstation
```

Generate a key pair

```bash
ssh-keygen
```

Prepare an empty folder for the configuration

```bash
mkdir terraform
```

And navigate to it

```bash
cd terraform
```

Open an empty file

```bash
vi main.tf
```

And paste the following configuration

```terraform
terraform {
  required_providers {
    lxd = {
      source = "terraform-lxd/lxd"
      version = "2.5.0"
    }
  }
}

provider "lxd" {
  generate_client_certificates = true
  accept_remote_certificate    = true

  remote {
    name     = "lxd"
    address  = "https://192.168.99.101:8443"
    password = "Parolka-12345"
    default  = true
  }
}

locals {
  cloud-init-config = <<EOF
#cloud-config
disable_root: false
ssh_authorized_keys:
  - ${file("~/.ssh/id_rsa.pub")}
package_upgrade: true
packages:
  - openssh-server
timezone: Europe/Sofia
EOF
}

resource "lxd_instance" "debian_instance" {
  name  = "mydebian"
  image = "images:debian/12/cloud"

  config = {
    "boot.autostart" = true
    "user.user-data" = local.cloud-init-config
  }

  device {
    name = "ssh"
    type = "proxy"
    properties = {
      listen = "tcp:0.0.0.0:10101"
      connect = "tcp:127.0.0.1:22"
    }
  }

  device {
    name = "http"
    type = "proxy"
    properties = {
      listen = "tcp:0.0.0.0:10102"
      connect = "tcp:127.0.0.1:80"
    }
  }

}

output "debian_instance_ip" {
  value = lxd_instance.debian_instance.ipv4_address
}
```

Save and close it

Initialize the workspace

```bash
terraform init
```

Check what Terraform will do

```bash
terraform plan
```

And apply the configuration

```bash
terraform apply
```

Once the process is done, try to open an SSH session to the new container

```bash
ssh -i ~/.ssh/id_rsa -p 10101 root@192.168.99.101
```

It works :)

Exit the container and return to the workstation machine

```bash
exit
```

## Ansible

We are ready to add some Ansible magic ;)

Go one level up

```bash
cd ..
```

Create another empty folder

```bash
mkdir ansible
```

Enter the new folder

```bash
cd ansible
```

Create an empty inventory file

```bash
vi inventory.ini
```

And paste the following content

```text
mydebian ansible_host=192.168.99.101 ansible_port=10101 ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Then create an empty configuration file

```bash
vi ansible.cfg
```

And paste the following content

```text
[defaults]
host_key_checking = False
inventory = inventory.ini
```

Finally, create an empty playbook file

```bash
vi playbook.yaml
```

And use the following content

```yaml
---
- hosts: all

  tasks:
    - name: Install NGINX
      apt:
        name: nginx
        state: present
        update_cache: yes
    - name: Set new index.html
      copy:
        dest: /var/www/html/index.html
        content: |
          It works, but with some Ansible magic :)
```

We can do a basic check

```bash
ansible-playbook --syntax-check playbook.yaml
```

See which hosts will be affected by the playbook (only one in our case)

```bash
ansible-playbook --list-hosts playbook.yaml
```

And see what tasks will be executed

```bash
ansible-playbook --list-tasks playbook.yaml
```

Finally, execute the playbook

```bash
ansible-playbook playbook.yaml
```

Once done, try to reach the NGINX instance that we just installed in the container

```bash
curl http://192.168.99.101:10102
```

It works like a charm :)

## Clean Up

Once done experimenting, we can clean up the bits one by one or destroy the whole environment

Let's do it piece by piece

Go in the Terraform folder

```bash
cd ../terraform
```

Destroy the container with Terraform

```bash
terraform destroy
```

Exit to the host

```bash
exit
```

And destroy the whole environment

```bash
vagrant destroy --force
```
