# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
    
  config.ssh.insert_key = false

  config.vm.define "workstation" do |workstation|
    workstation.vm.box = "shekeriev/debian-12.11"
    workstation.vm.hostname = "workstation.batmitko.lab"
    workstation.vm.network "private_network", ip: "192.168.99.100"
    workstation.vm.provision "shell", inline: <<EOS
export DEBIAN_FRONTEND=noninteractive

echo "* Install prerequisites ..."
apt-get update
apt-get install -y git gpg tree vim tar zip unzip ca-certificates curl

echo "* Install Ansible Core ..."
UBUNTU_CODENAME=jammy
wget -q -O- "https://keyserver.ubuntu.com/pks/lookup?fingerprint=on&op=get&search=0x6125E2A8C77F2818FB7BD15B93C4A3FD7BB9C367" | gpg --dearmour -o /usr/share/keyrings/ansible-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/ansible-archive-keyring.gpg] http://ppa.launchpad.net/ansible/ansible/ubuntu $UBUNTU_CODENAME main" | tee /etc/apt/sources.list.d/ansible.list
apt-get update && apt-get install -y ansible

echo "* Install Terraform ..."
wget -q -O - https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list
apt-get update && apt-get install -y terraform
EOS
  end
  
  config.vm.define "lxd" do |lxd|
    lxd.vm.box = "shekeriev/debian-12.11"
    lxd.vm.hostname = "lxd.batmitko.lab"
    lxd.vm.network "private_network", ip: "192.168.99.101"
    lxd.vm.provision "shell", inline: <<EOS
export DEBIAN_FRONTEND=noninteractive

echo "* Install LXD ..."
apt-get update
apt-get install -y curl lxd

echo "* Initialize LXD ..."
lxd init --preseed <<EOF
config:
  core.https_address: '[::]:8443'
  core.trust_password: Parolka-12345
networks:
- config:
    ipv4.address: auto
    ipv6.address: none
  description: ""
  name: lxdbr0
  type: ""
  project: default
storage_pools:
- config: {}
  description: ""
  name: default
  driver: dir
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      network: lxdbr0
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
projects: []
cluster: null
EOF

echo "* Add the vagrant user to the group ..."
usermod -aG lxd vagrant

echo "* Patch the images: remote catalog ..."
lxc remote set-url images https://images.lxd.canonical.com
su vagrant -c 'lxc remote set-url images https://images.lxd.canonical.com'
EOS
  end
  
end