$init = <<-SCRIPT
export DEBIAN_FRONTEND=noninteractive

export KUBE_VERSION=1.11.3
export DOCKER_VERSION=17.03.3

# Install necessary packages
apt-get update && \
    apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

# Add repo keys
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

# Add repo for docker and k8s
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
add-apt-repository \
    "deb [arch=amd64] http://apt.kubernetes.io/ \
    kubernetes-$(lsb_release -cs) \
    main"

# Install docker and k8s
apt-get update && \
    apt-get install -y docker-ce=$DOCKER_VERSION~ce-0~ubuntu-$(lsb_release -cs) kubelet=$KUBE_VERSION-00 kubeadm=$KUBE_VERSION-00 kubectl=$KUBE_VERSION-00 && \
    apt-mark hold docker-ce kubelet kubeadm kubectl

# Install kubectl autocomplete
apt-get update && \
    apt-get install -y bash-completion && \
    echo -e "source /etc/bash_completion\nsource <(kubectl completion bash)\nsource /etc/bash_completion" >> ~/.bashrc

# Turn of swap, because this is not supported by k8s
swapoff -a

# Pull down all necessary k8s container images
kubeadm config images pull
docker pull nginx:1.15.3

SCRIPT

Vagrant.configure("2") do |config|

  # Install Docker, and kubernetes along with kubeadm.
  config.vm.provision "shell", inline: $init

  ## Set memory and cpu usage
  config.vm.provider "virtualbox" do |vbox|
    vbox.cpus = 1
    vbox.memory = 1024
  end

  # K8S Master
  config.vm.define "kube-master" do |config|
    config.vm.box = "ubuntu/xenial64"
    config.vm.hostname = "kube-master"

    config.vm.network "private_network", ip: "192.168.79.10"
    config.vm.network :forwarded_port, guest: 8001, host: 8001
  end

  # K8S Node01
  config.vm.define "kube-agent01" do |config|
    config.vm.box = "ubuntu/xenial64"
    config.vm.hostname = "kube-agent01"

    config.vm.network "private_network", ip: "192.168.79.20"
  end

  # K8S Node01
  config.vm.define "kube-agent02" do |config|
    config.vm.box = "ubuntu/xenial64"
    config.vm.hostname = "kube-agent02"

    config.vm.network "private_network", ip: "192.168.79.21"
  end

end
