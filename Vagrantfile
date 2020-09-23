# -*- mode: ruby -*-
# vi: set ft=ruby :

# Fix for missing repo error, detailed and fixed
# https://github.com/dotless-de/vagrant-vbguest/issues/367#issuecomment-619375784

# https://github.com/dotless-de/vagrant-vbguest/issues/367
# https://github.com/dotless-de/vagrant-vbguest/pull/373
if defined?(VagrantVbguest)
  class MyWorkaroundInstallerUntilPR373IsMerged < VagrantVbguest::Installers::CentOS
    protected
    
    def has_rel_repo?
      unless instance_variable_defined?(:@has_rel_repo)
        rel = release_version
        @has_rel_repo = communicate.test(centos_8? ? 'yum repolist' : "yum repolist --enablerepo=C#{rel}-base --enablerepo=C#{rel}-updates")
      end
      @has_rel_repo
    end

    def centos_8?
      release_version && release_version.to_s.start_with?('8')
    end

    def install_kernel_devel(opts=nil, &block)
      if centos_8?
        communicate.sudo('yum update -y kernel', opts, &block)
        communicate.sudo('yum install -y kernel-devel', opts, &block)
        communicate.sudo('shutdown -r now', opts, &block)

        begin
          sleep 10
        end until @vm.communicate.ready?
      else
        rel = has_rel_repo? ? release_version : '*'
        cmd = "yum install -y kernel-devel-`uname -r` --enablerepo=C#{rel}-base --enablerepo=C#{rel}-updates"
        communicate.sudo(cmd, opts, &block)
      end
    end
  end
end

Vagrant.configure('2') do |config|
  config.vm.hostname = 'controller1'

  config.vm.network 'private_network', ip: '192.168.33.3', auto_config: false

  config.vbguest.auto_update = true

  config.vm.box = 'centos/8'

  # Explicitly set VM Box URL
  config.vm.box_url = 'https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-Vagrant-8.2.2004-20200611.2.x86_64.vagrant-virtualbox.box'

  # Add fix
  if defined?(MyWorkaroundInstallerUntilPR373IsMerged)
    config.vbguest.installer = MyWorkaroundInstallerUntilPR373IsMerged
  end

  # The default CentOS box comes with a root disk that is too small to fit a
  # deployment on so we need to make it bigger.
  config.disksize.size = '30GB'

  config.vm.provider 'virtualbox' do |vb|
    vb.memory = '8192'
    vb.cpus = '6'
    vb.linked_clone = true
  end

  config.vm.provider 'vmware_fusion' do |vmware|
    vmware.vmx['memsize'] = '4096'
    vmware.vmx['vhv.enable'] = 'TRUE'
    vmware.linked_clone = true
  end

  config.vm.provision 'shell', inline: <<-SHELL

    # Extend the root disk
    sudo parted "/dev/sda" ---pretend-input-tty <<EOF
resizepart
1
100%
Yes
-0
quit
EOF
    sudo xfs_growfs -d /

    echo "cat > /etc/selinux/config << EOF
SELINUX=disabled
SELINUXTYPE=targeted
EOF" | sudo -s
      cat /etc/selinux/config
  SHELL

  # NOTE: Reboot to apply selinux change, requires the reload plugin:
  #   vagrant plugin install vagrant-reload
  config.vm.provision :reload

  config.vm.provision 'shell', privileged: false, inline: <<-SHELL
    cat << EOF | sudo tee /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
USERCTL=no
BOOTPROTO=none
IPADDR=192.168.33.3
NETMASK=255.255.255.0
ONBOOT=yes
EOF
    sudo ifup eth1

    /vagrant/dev/install-dev.sh

    # Configure the legacy development environment. This has been retained
    # while transitioning to the new development environment.
    cat > /vagrant/kayobe-env << EOF
export KAYOBE_CONFIG_PATH=/vagrant/etc/kayobe
export KOLLA_CONFIG_PATH=/vagrant/etc/kolla
EOF
    cp /vagrant/dev/dev-vagrant.yml /vagrant/etc/kayobe/
    cp /vagrant/dev/dev-hosts /vagrant/etc/kayobe/inventory
    cp /vagrant/dev/dev-vagrant-network-allocation.yml /vagrant/etc/kayobe/network-allocation.yml
  SHELL
end
