# -*- mode: ruby -*-
# vi: set ft=ruby :

NUM_MASTER_NODE = 1
NUM_WORKER_NODE = 3

IP_NW = "192.168.56."
MASTER_IP_START = 20
NODE_IP_START = 30

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/mantic64"
  config.vm.boot_timeout = 360

  (1..NUM_MASTER_NODE).each do |i|
      config.vm.define "kubemaster0#{i}" do |node|
        # Name shown in the GUI
        node.vm.provider "virtualbox" do |vb|
            vb.name = "kubemaster0#{i}"
            vb.memory = 3072
            vb.cpus = 2
        end
        node.vm.hostname = "kubemaster0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"
        node.vm.network "forwarded_port", guest: 22, host: "#{2710 + i}"
        node.vm.cloud_init content_type: "text/cloud-config", path: "./cloud-init.yaml"
      end
  end

  # Provision Worker Nodes
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "kubenode0#{i}" do |node|
        node.vm.provider "virtualbox" do |vb|
            vb.name = "kubenode0#{i}"
            vb.memory = 2048
            vb.cpus = 2
        end
        node.vm.hostname = "kubenode0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{NODE_IP_START + i}"
        node.vm.network "forwarded_port", guest: 22, host: "#{2720 + i}"
        node.vm.cloud_init content_type: "text/cloud-config", path: "./cloud-init.yaml"
    end
  end
end
