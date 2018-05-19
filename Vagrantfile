# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  (1..4).each do |i|
    config.vm.define "k8s-#{i}" do |s|
      s.vm.box = "ubuntu/xenial64"
      s.vm.hostname = "k8s-#{i}"

      if i == 1 then
        s.vm.network :forwarded_port, host: 8001, guest: 8001
      end
      s.vm.network :private_network, ip: "172.42.42.#{i+10}", netmask: "255.255.255.0",
                   auto_config: true,
                   virtualbox__intnet: "k8s-net"

      s.vm.provider "virtualbox" do |v|
        v.name = "k8s-#{i}"
        if i == 1 then
          v.memory = 2048 # master require much memory
        else
          v.memory = 1024
        end
        v.gui = false
      end

      # https://kubernetes.io/docs/setup/independent/install-kubeadm/
      # https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
      s.vm.provision "shell", inline: <<-EOFP
echo "update"
apt-get update
apt-get install language-pack-UTF-8

echo "install docker"
apt-get install -y docker.io

echo "install kubeadm, kubelet and kubectl"
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

echo "Common setup finished, you can see README.md to continue next works."
EOFP
    end
  end
end
