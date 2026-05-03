BUILD_MODE = "NAT"

NUM_MASTER_NODES = 3
NUM_WORKER_NODES = 3

# Network parameters for NAT mode
IP_NW = "192.168.56"
MASTER_IP_START = 40
NODE_IP_START = 60

# Host operating sysem detection
module OS
  def OS.windows?
    (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
  end

  def OS.mac?
    (/darwin/ =~ RUBY_PLATFORM) != nil
  end

  def OS.unix?
    !OS.windows?
  end

  def OS.linux?
    OS.unix? and not OS.mac?
  end

  def OS.jruby?
    RUBY_ENGINE == "jruby"
  end
end

# Determine host adpater for bridging in BRIDGE mode
def get_bridge_adapter()
  if OS.windows?
    return %x{powershell -Command "Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Get-NetAdapter | Select-Object -ExpandProperty InterfaceDescription"}.chomp
  elsif OS.linux?
    return %x{ip route | grep default | awk '{ print $5 }'}.chomp
  elsif OS.mac?
    return %x{mac/mac-bridge.sh}.chomp
  end
end

def get_machine_id(vm_name)
  machine_id_filepath = ".vagrant/machines/#{vm_name}/virtualbox/id"
  if not File.exist? machine_id_filepath
    return nil
  else
    return File.read(machine_id_filepath)
  end
end

# Helper method to determine whether all nodes are up
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

# Sets up hosts file and DNS
def setup_dns(node)
  # Set up /etc/hosts
  node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/vagrant/setup-hosts.sh" do |s|
    s.args = [IP_NW, BUILD_MODE, NUM_WORKER_NODES, MASTER_IP_START, NODE_IP_START]
  end
  # Set up DNS resolution
  node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
end

# Runs provisioning steps that are required by masters and workers
def provision_kubernetes_node(node)

  node.vm.provision "setup-ssh", :type => "shell", :path => "ubuntu/ssh.sh"
end

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/jammy64"
  config.vm.boot_timeout = 900

  config.vm.box_check_update = false

# Provision Master Nodes
(1..NUM_MASTER_NODES).each do |i|

  config.vm.define "master0#{i}" do |node|

    node.vm.provider "virtualbox" do |vb|
      vb.name = "master0#{i}"
      vb.memory = 2048
      vb.cpus = 2
    end

    node.vm.hostname = "master0#{i}"

    if BUILD_MODE == "BRIDGE"
      node.vm.network :public_network, bridge: get_bridge_adapter()
    else
      node.vm.network :private_network,
        ip: IP_NW + ".#{MASTER_IP_START + i}"

      node.vm.network "forwarded_port",
        guest: 22,
        host: "#{2710 + i}"
    end

    provision_kubernetes_node node

    node.vm.provision "file",
      source: "./ubuntu/tmux.conf",
      destination: "$HOME/.tmux.conf"

    node.vm.provision "file",
      source: "./ubuntu/vimrc",
      destination: "$HOME/.vimrc"
  end
end

  # Provision Worker Nodes
  (1..NUM_WORKER_NODES).each do |i|
    config.vm.define "node0#{i}" do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "node0#{i}"
        vb.memory = 1024
        vb.cpus = 1
      end
      node.vm.hostname = "node0#{i}"
      if BUILD_MODE == "BRIDGE"
        node.vm.network :public_network, bridge: get_bridge_adapter()
      else
        node.vm.network :private_network, ip: IP_NW + ".#{NODE_IP_START + i}"
        node.vm.network "forwarded_port", guest: 22, host: "#{2720 + i}"
      end
      provision_kubernetes_node node
    end
  end

  if BUILD_MODE == "BRIDGE"

    config.trigger.after :up do |trigger|
      trigger.name = "Post provisioner"
      trigger.ignore = [:destroy, :halt]
      trigger.ruby do |env, machine|
        if all_nodes_up()
          puts "    Gathering IP addresses of nodes..."
          nodes = []
          (1..NUM_MASTER_NODES).each do |i|
            nodes.push("master0#{i}")
          end
          ips = []
          (1..NUM_WORKER_NODES).each do |i|
            nodes.push("node0#{i}")
          end
          nodes.each do |n|
            ips.push(%x{vagrant ssh #{n} -c 'public-ip'}.chomp)
          end
          hosts = ""
          ips.each_with_index do |ip, i|
            hosts << ip << "  " << nodes[i] << "\n"
          end
          puts "    Setting /etc/hosts on nodes..."
          File.open("hosts.tmp", "w") { |file| file.write(hosts) }
          nodes.each do |node|
            system("vagrant upload hosts.tmp /tmp/hosts.tmp #{node}")
            system("vagrant ssh #{node} -c 'cat /tmp/hosts.tmp | sudo tee -a /etc/hosts'")
          end
          File.delete("hosts.tmp")
          puts <<~EOF

                 VM build complete!

                 Use either of the following to access any NodePort services you create from your browser
                 replacing "port_number" with the number of your NodePort.

               EOF
          (1..NUM_WORKER_NODES).each do |i|
            puts "  http://#{ips[i]}:port_number"
          end
          puts ""
        else
          puts "    Nothing to do here"
        end
      end
    end
  end
end
