# Portions of this Vagrantfile are based on the vbox file provided by
# https://github.com/kodekloudhub/certified-kubernetes-administrator-course
#
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

# Check required plugins
REQUIRED_PLUGINS_LIBVIRT = %w(vagrant-libvirt)
exit unless REQUIRED_PLUGINS_LIBVIRT.all? do |plugin|
  Vagrant.has_plugin?(plugin) || (
    puts "The #{plugin} plugin is required. Please install it with:"
    puts "$ vagrant plugin install #{plugin}"
    false
  )
end

Vagrant.require_version ">= 2.0.2"


NUM_MASTER_NODE = 1
NUM_WORKER_NODE = 2

Vagrant.configure("2") do |config|
    config.ssh.forward_agent = true
    config.vm.box = "generic/ubuntu2004"

    (1..NUM_MASTER_NODE).each do |i|
      config.vm.define "kubemaster-0#{i}" do |node|
        node.vm.box = "generic/ubuntu2004"
        node.vm.hostname = "kubemaster-0#{i}"
        node.vm.synced_folder ".", "/vagrant", disabled: true

        node.vm.provider :libvirt do |v|
          v.memory = 4096
          v.cpus = 4
        end

        node.vm.network :private_network,
                        :ip => "192.168.1.1#{i}",
                        :libvirt__netmask => '255.255.0.0',
                        :libvirt__network_name => 'pod_network',
                        :libvirt__forward_mode => 'none'

        node.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false
        node.vm.provision :shell, privileged: true, inline: $install_tools
      end
    end

    # Provision Worker Nodes
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "kubenode-0#{i}" do |node|
      node.vm.box = "generic/ubuntu2004"
      node.vm.hostname = "kubenode-0#{i}"
      node.vm.synced_folder ".", "/vagrant", disabled: true

      node.vm.provider :libvirt do |v|
          v.memory = 4096
          v.cpus = 4
      end
      # node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"
      node.vm.network :private_network,
                        :ip => "192.168.2.1#{i}",
                        :libvirt__netmask => '255.255.0.0',
                        :libvirt__network_name => 'pod_network',
                        :libvirt__forward_mode => 'none'

      node.vm.provision :shell, privileged: false, inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;"
      node.vm.provision :shell, privileged: true, inline: $install_tools
    end
  end
end

$install_tools = <<-SCRIPT
# Disable AAAA records; speeds up APT for v4 only networks
echo -e "precedence ::ffff:0:0/96  100" >> /etc/gai.conf
DEBIAN_FRONTEND=noninteractive apt-get update > /dev/null 2>&1
DEBIAN_FRONTEND=noninteractive apt-get install -y chrony htop git > /dev/null 2>&1
#sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -qy > /dev/null 2>&1
SCRIPT
