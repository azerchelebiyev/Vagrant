BUILD_MODE = "NAT"

NUM_MASTER_NODES = 1
NUM_WORKER_NODES = 1

# Network parameters
IP_NW = "192.168.56"
MASTER_IP_START = 40
NODE_IP_START = 60

# Determine whether all nodes are up
def get_machine_id(vm_name)
  machine_id_filepath = ".vagrant/machines/#{vm_name}/virtualbox/id"

  if !File.exist?(machine_id_filepath)
    return nil
  else
    return File.read(machine_id_filepath)
  end
end

def all_nodes_up()
  (1..NUM_MASTER_NODES).each do |i|
    if get_machine_id("master0#{i}").nil?
      return false
    end
  end

  (1..NUM_WORKER_NODES).each do |i|
    if get_machine_id("node0#{i}").nil?
      return false
    end
  end

  return true
end

# Configure hosts file and DNS
def setup_dns(node)
  node.vm.provision "setup-hosts",
    type: "shell",
    path: "ubuntu/vagrant/setup-hosts.sh" do |s|

    s.args = [
      IP_NW,
      BUILD_MODE,
      NUM_WORKER_NODES,
      MASTER_IP_START,
      NODE_IP_START
    ]
  end

  node.vm.provision "setup-dns",
    type: "shell",
    path: "ubuntu/update-dns.sh"
end

# Common provisioning for master and worker nodes
def provision_kubernetes_node(node)
  node.vm.provision "setup-ssh",
    type: "shell",
    path: "ubuntu/ssh.sh"
end

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/jammy64"

  config.vm.boot_timeout = 900

  config.vm.box_check_update = false

  # Master Nodes
  (1..NUM_MASTER_NODES).each do |i|

    config.vm.define "master0#{i}" do |node|

      node.vm.provider "virtualbox" do |vb|
        vb.name = "master0#{i}"
        vb.memory = 2048
        vb.cpus = 2
      end

      node.vm.hostname = "master0#{i}"

      node.vm.network :private_network,
        ip: IP_NW + ".#{MASTER_IP_START + i}"

      node.vm.network "forwarded_port",
        guest: 22,
        host: 2710 + i

      setup_dns node
      provision_kubernetes_node node

      node.vm.provision "file",
        source: "./ubuntu/tmux.conf",
        destination: "$HOME/.tmux.conf"

      node.vm.provision "file",
        source: "./ubuntu/vimrc",
        destination: "$HOME/.vimrc"
    end
  end

  # Worker Nodes
  (1..NUM_WORKER_NODES).each do |i|

    config.vm.define "node0#{i}" do |node|

      node.vm.provider "virtualbox" do |vb|
        vb.name = "node0#{i}"
        vb.memory = 1024
        vb.cpus = 1
      end

      node.vm.hostname = "node0#{i}"

      node.vm.network :private_network,
        ip: IP_NW + ".#{NODE_IP_START + i}"

      node.vm.network "forwarded_port",
        guest: 22,
        host: 2720 + i

      setup_dns node
      provision_kubernetes_node node
    end
  end
end