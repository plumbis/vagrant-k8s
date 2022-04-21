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


Vagrant.configure("2") do |config|
    config.ssh.forward_agent = true

    (1..3).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = "generic/ubuntu2004"
            node.vm.hostname = "node-#{i}"
            node.vm.synced_folder ".", "/vagrant", disabled: true

            node.vm.provider :libvirt do |v|
                v.memory = 4096
                v.cpus = 4
            end

            node.vm.network "private_network",
            :mac => "44:38:39:00:00:3#{i}",
            :libvirt__tunnel_type => 'udp',
            :libvirt__tunnel_local_ip => '127.0.0.1',
            :libvirt__tunnel_local_port => "#{ 8030 + i }",
            :libvirt__tunnel_ip => '127.0.0.1',
            :libvirt__tunnel_port => "#{ 9030 + i }",
            :libvirt__iface_name => 'eth1',
            auto_config: false

            node.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false
            node.vm.provision :shell, privileged: true, inline: $install_tools

        end
    end
end
$install_tools = <<-SCRIPT
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update > /dev/null 2>&1
sudo apt-get install -y chrony lldpd htop git > /dev/null 2>&1
sudo apt-get upgrade -qy > /dev/null 2>&1
echo "configure lldp portidsubtype ifname" > /etc/lldpd.d/port_info.conf
SCRIPT
