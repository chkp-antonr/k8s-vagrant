IP_NET="192.168.3"
IP_START=70
DEFAULT_MEM=2048
WORKER_MEM=3072
DEFAULT_CPU=2
WORKER_CPU=2

Vagrant.configure("2") do |config|
  config.vm.provider :virtualbox do |vb|
      vb.memory = "#{DEFAULT_MEM}"
      vb.cpus = "#{DEFAULT_CPU}"
  end

  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.box = "ubuntu/focal64"
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: "#{IP_NET}.#{IP_START}"
    master.vm.provision :shell, privileged: false, inline: $provision_master_node
  end

  %w{worker1 worker2}.each_with_index do |name, i|
    config.vm.define name do |worker|
      worker.vm.provider :virtualbox do |vb|
        vb.memory = "#{WORKER_MEM}"
        vb.cpus = "#{WORKER_CPU}"
      end
      worker.vm.box = "ubuntu/focal64"
      worker.vm.hostname = name
      worker.vm.network :private_network, ip: "#{IP_NET}.#{i + 1 + IP_START}"
      worker.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo /vagrant/join.sh
#echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=#{IP_NET}.#{i + 21}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=#{IP_NET}.#{i + 1 + IP_START}"'
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=#{IP_NET}.#{i + 1 + IP_START}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


$install_common_tools = <<-SCRIPT
# bridged traffic to iptables is enabled for kube-router.
cat >> /etc/ufw/sysctl.conf <<EOF
net/bridge/bridge-nf-call-ip6tables = 1
net/bridge/bridge-nf-call-iptables = 1
net/bridge/bridge-nf-call-arptables = 1
EOF

# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl enable docker.service --now
sudo usermod -aG docker $(whoami)

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
#apt-get -qq install -y kubernetes-cni=0.6.0-00 kubelet=1.19.6-00 kubectl=1.19.6-00 kubeadm=1.19.6-00
apt-get install -qy kubelet=1.20.4 kubectl=1.20.4 kubeadm=1.20.4 kubernetes-cni
SCRIPT
# curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'

$provision_master_node = <<-SHELL
OUTPUT_FILE=/vagrant/join.sh
INIT_FILE=/vagrant/kubeadm_init.log
rm -rf $OUTPUT_FILE

# install helm
wget https://get.helm.sh/helm-v3.6.1-linux-amd64.tar.gz
tar -zxvf helm-v3.6.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

# Start cluster
#sudo kubeadm init --apiserver-advertise-address=#{IP_NET}.#{IP_START} --pod-network-cidr=10.253.192.0/18 | grep "kubeadm join" > ${OUTPUT_FILE}
sudo kubeadm init --apiserver-advertise-address=#{IP_NET}.#{IP_START} --pod-network-cidr=10.253.192.0/18  > ${INIT_FILE}
grep "kubeadm join" ${INIT_FILE}  > ${OUTPUT_FILE}
grep "discovery-token" ${INIT_FILE} >> ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=#{IP_NET}.#{IP_START}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure flannel
#curl -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
#sed -i.bak 's|"/opt/bin/flanneld",|"/opt/bin/flanneld", "--iface=enp0s8",|' kube-flannel.yml
#kubectl create -f kube-flannel.yml

# or Calico
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml


sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL
