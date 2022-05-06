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

# NUM_CONTROLLER_NODE = 1
NUM_WORKER_NODE = 2

Vagrant.configure("2") do |config|
  config.ssh.forward_agent = true
  config.vm.box = "generic/ubuntu2004"

  # (1..NUM_CONTROLLER_NODE).each do |i|
  (1..1).each do |i|
    config.vm.define "kubecontroller-0#{i}" do |node|
      node.vm.hostname = "kubecontroller-0#{i}"
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

      node.vm.provision :shell, privileged: true, inline: $configure_linux
      node.vm.provision :shell, privileged: true, inline: $install_docker
      node.vm.provision :shell, privileged: true, inline: $install_tools
      node.vm.provision :shell, privileged: true, inline: $configure_kube_controller

    end
  end

    # Provision Worker Nodes
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "kubenode-0#{i}" do |node|
      node.vm.hostname = "kubenode-0#{i}"
      node.vm.synced_folder ".", "/vagrant", disabled: true

      node.vm.provider :libvirt do |v|
        v.memory = 4096
        v.cpus = 4
      end

      node.vm.network :private_network,
                        :ip => "192.168.2.1#{i}",
                        :libvirt__netmask => '255.255.0.0',
                        :libvirt__network_name => 'pod_network',
                        :libvirt__forward_mode => 'none'

      node.vm.provision :shell, privileged: true, inline: $configure_linux
      node.vm.provision :shell, privileged: true, inline: $install_docker
      node.vm.provision :shell, privileged: true, inline: $install_tools
      node.vm.provision :shell, privileged: true, inline: $configure_kube_worker
    end
  end
end

$configure_linux = <<-SCRIPT
echo "Applying Linux system settings"

# Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;

# Disable AAAA records; speeds up APT for v4 only networks
echo -e "precedence ::ffff:0:0/96  100" >> /etc/gai.conf

# Disable swap to prevent kubeadm warnings
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Enable k8s related networking settings
modprobe br_netfilter
echo -e "br_netfilter" > /etc/modules-load.d/k8s.conf
echo -e "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/k8s.conf
echo -e "net.bridge.bridge-nf-call-iptables = 1"  >> /etc/sysctl.d/k8s.conf
sed -i 's/^#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.d/99-sysctl.conf

# Revert systemd change that breaks sysctl command (commit https://github.com/systemd/systemd-stable/commit/5d4fc0e665a3639f92ac880896c56f9533441307#)
# Fix is in procps https://gitlab.com/procps-ng/procps/-/issues/191
sed 's/net.ipv4.conf.\*.promote_secondaries = 1//' /usr/lib/sysctl.d/50-default.conf > /usr/lib/sysctl.d/50-default.conf
sed 's/-net.ipv4.conf.all.promote_secondaries//' /usr/lib/sysctl.d/50-default.conf > /usr/lib/sysctl.d/50-default.conf

sudo sysctl --system > /dev/null 2>&1
SCRIPT

$install_tools = <<-SCRIPT
echo "Installing additional software packages"
DEBIAN_FRONTEND=noninteractive apt-get install -y chrony htop git > /dev/null 2>&1
SCRIPT

$install_docker = <<-SCRIPT
echo "Installing containerd and Kubeadm"

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | DEBIAN_FRONTEND=noninteractive apt-key add - > /dev/null 2>&1
echo deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable > /etc/apt/sources.list.d/docker.list

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | DEBIAN_FRONTEND=noninteractive apt-key add - > /dev/null 2>&1
echo "deb [arch=amd64] https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

kube_ver=1.23.6-00
DEBIAN_FRONTEND=noninteractive apt-get update > /dev/null 2>&1
DEBIAN_FRONTEND=noninteractive apt-get install -y kubeadm=$kube_ver kubelet=$kube_ver kubectl=$kube_ver containerd.io > /dev/null 2>&1
DEBIAN_FRONTEND=noninteractive apt-mark hold kubelet kubeadm kubectl > /dev/null 2>&1

# Remove the stock containerd config file and let it auto generate
rm -f /etc/containerd/config.toml
systemctl restart containerd.service

SCRIPT

$configure_kube_controller = <<-SCRIPT
# CLI magic to pull the IP address for future multi-controller setups ip -f inet addr show eth1 | sed -En -e 's/.*inet ([0-9.]+).*/\1/p'
kubeadm init --apiserver-advertise-address 192.168.1.11 --pod-network-cidr 10.244.0.0/16 > /dev/null 2>&1

# Generate a known token for workers to join the cluster.
kubeadm token create 123456.abcdefghijklmnop > /dev/null 2>&1

# Enable kubectl for root and vagrant users
mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown root:root /root/.kube/config
mkdir -p /home/vagrant/.kube
cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown vagrant:vagrant /home/vagrant/.kube/config

# Install Weave
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')" > /dev/null 2>&1

SCRIPT

$configure_kube_worker = <<-SCRIPT
# Join the worker to the cluster
kubeadm join 192.168.1.11:6443 --token 123456.abcdefghijklmnop --discovery-token-unsafe-skip-ca-verification > /dev/null 2>&1

SCRIPT