IMAGE_NAME = "bento/ubuntu-20.04"
N = 2

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false
    config.vm.provision :shell, privileged: true, inline: $install_common_tools       

  
        config.vm.define "k8s-master" do |master|
	        master.vm.provider "virtualbox" do |v|
              v.memory = 2048
              v.cpus = 2	
	    	end
            master.vm.box = IMAGE_NAME
            master.vm.network "private_network", ip: "192.168.50.10"
            master.vm.hostname = "k8s-master"
            master.vm.provision :shell, privileged: false, inline: $provision_master_node        
            
        end
    
    (1..N).each do |i|

        config.vm.define "node-#{i}" do |node|
		    node.vm.provider "virtualbox" do |v|
              v.memory = 1024
              v.cpus = 2
		    end
			node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
            node.vm.hostname = "node-#{i}"
       end
    end
	
end

$install_common_tools = <<-SCRIPT
#!/bin/bash

set -euxo pipefail

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# install prerequistes
sudo apt-get -qq update 
sudo apt-get -qq install -y apt-transport-https ca-certificates curl gnupg


# Install docker
for pkg in docker.io docker-doc docker-compose containerd runc; do sudo apt-get remove -y $pkg; done

echo "*****************Install preq for docker*****************"
#sudo apt-get -qq  update 
sudo apt-get install -y ca-certificates curl gnupg 
sudo install -m 0755 -d /etc/apt/keyrings 

echo "***************** Gpg *****************"
if [ ! -e /etc/apt/keyrings/docker.gpg  ]
    then
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg  --dearmor -o /etc/apt/keyrings/docker.gpg 
        sudo chmod a+r /etc/apt/keyrings/docker.gpg
fi

echo   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


sudo apt-get -qq update 
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 

# install kubernetes
pkgs='kubeadm'

sudo usermod -a -G docker vagrant

if [ ! -e /etc/apt/keyrings/kubernetes-apt-keyring.gpg ]
    then
         curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
fi

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


echo " *****************Installation de kubeadm, kubelet, kubectl*****************"
sudo apt-get -qq update  
sudo apt-get -qq  install -y kubelet kubeadm kubectl
#disabled update for these packages

# disabled swap
echo " *****************disabled swap*****************"
sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

SCRIPT

$provision_master_node = <<-SHELL
echo " *****************restart docker*****************"
sudo systemctl restart docker
sudo systemctl daemon-reload
sudo kubeadm reset -f

echo " *****************init kubadm*****************"
if [ -e /etc/containerd/config.toml ]
    then
    sudo mv /etc/containerd/config.toml /tmp
    sudo systemctl restart containerd
    sudo kubeadm init --apiserver-advertise-address="192.168.50.10" --apiserver-cert-extra-sans="192.168.50.10"  --node-name k8s-master --pod-network-cidr=192.168.0.0/16
fi

SHELL

$configure_network = <<SHELL
#!/bin/bash

set -euxo pipefail

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
SHELL

$poste_install = <<-SHELL


if  dpkg -s $pkgs >/dev/null 2>&1
    then   
    sudo apt-mark unhold kubeadm kubelet
    echo "************unhold kubeadm kubelet******************"
fi

sudo apt-mark hold kubelet kubeadm kubectl
sudo mkdir -p /home/vagrant/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
sudo chown vagrant:vagrant /home/vagrant/.kube/config
SHELL