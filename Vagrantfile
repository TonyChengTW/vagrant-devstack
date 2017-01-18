# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Edit by Tony Cheng 
# Email : tonycheng@cloudcube.com.tw
#

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2" if not defined? VAGRANTFILE_API_VERSION

conf = {}

require 'yaml'
conf_path = ENV.fetch('USER_CONF','vagrantfile-config.yaml')
if File.file?(conf_path)
    user_conf = YAML.load_file(conf_path)
    conf.update(user_conf)
else
    raise "Configuration file #{conf_path} does not exist."
end

def config_vb_network(vm, conf)
    # this will be the endpoint
    vm.network :private_network, ip: conf['internal1_ip'], :netmask => conf['internal1_netmask']
    # this will be the OpenStack "public" network ip and subnet mask
    # should match floating_ip_range var in devstack.yml
    vm.network :private_network, ip: conf['external1_ip'], :netmask => conf['external1_netmask'], :auto_config => false
    vm.network :forwarded_port, guest: conf['vm_forward_port'], host: conf['host_forward_port']
end

def config_vb_provider(vm, conf)
    vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", conf['vm_memory']]
        vb.customize ["modifyvm", :id, "--cpus", conf['vm_cpus']]
       	# eth2 must be in promiscuous mode for floating IPs to be accessible
       	vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end
end

def config_lv_provider(vm, conf)
    vm.provider :libvirt do |libvirt|
        libvirt.host = conf['libvirt_host']
        libvirt.driver = conf['libvirt_driver']
        libvirt.username = conf['libvirt_username']
        libvirt.connect_via_ssh = conf['libvirt_connect_via_ssh']
        libvirt.storage_pool_name = conf['libvirt_storage_pool_name']
        libvirt.nested = conf['libvirt_nested']
    end
end

# -----------------  Box Start -------------------------------------------------------
def config_lv_define_box1(vm, conf)
  vm.define conf['hostname_box1'] do |box1|
    box1.vm.hostname = conf['hostname_box1']
    box1.vm.box = conf['imagename_box1']
#    box1.vm.network :private_network,
#                                :libvirt__network_name => "mgmt",
#                                :mac => conf['libvirt_mgmt_mac_box1'],
#                                :ip => conf['libvirt_mgmt_ip_box1'],
#                                :libvirt__netmask => conf['libvirt_mgmt_netmask_box1'],
#                                :libvirt__dhcp_enabled => false,
#                                :autostart => true
    box1.vm.network :public_network,
                                :network_name => "ext",
                                :ip => conf['libvirt_ext_ip_box1'],
                                :netmask => conf['libvirt_ext_netmask_box1'],
                                :gateway => conf['libvirt_ext_gateway_box1'],
                                :mac => conf['libvirt_ext_mac_box1'],
                                :dev => conf['libvirt_dev'],
                                :type => conf['libvirt_type'],
                                :mode => conf['libvirt_mode']
    box1.vm.network :private_network,
                                :libvirt__network_name => "ceph",
                                :mac => conf['libvirt_ceph_mac_box1'],
                                :ip => conf['libvirt_ceph_ip_box1'],
                                :libvirt__netmask => conf['libvirt_ceph_netmask_box1'],
                                :libvirt__dhcp_enabled => false,
                                :autostart => true
    box1.vm.network :private_network,
                                :libvirt__network_name => "vm_tunnel",
                                :mac => conf['libvirt_tunnel_mac_box1'],
                                :ip => conf['libvirt_tunnel_ip_box1'],
                                :libvirt__netmask => conf['libvirt_tunnel_netmask_box1'],
                                :libvirt__dhcp_enabled => false,
                                :autostart => true
    box1.vm.provider :libvirt do |domain|
      domain.memory = conf['memory_box1']
      domain.cpus = conf['cpus_box1']
      domain.management_network_name = 'vagrantmgmt_box1'
      domain.management_network_address = conf['libvirt_vagrantmgmt_ip_box1']
      domain.management_network_mode = conf['libvirt_mgmt_mode']
    end
    config_provision(box1.vm, conf)
  end
end
#  --------------  Box End -------------------------------------------------------

#  --------------  Provision Start ----------------------------------------------------
def config_provision(vm, conf)
    vm.provision :shell, run: "always", inline: "setenforce 0"
    vm.provision :shell, run: "always", inline: "sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config"
    vm.provision :shell, run: "always", inline: "yum -y install net-tools expect"
    vm.provision :shell, run: "always", path: "setrootpasswd.sh"
    vm.provision :shell, run: "always", inline: "ifup eth1"
    vm.provision :shell, run: "always", inline: "eval `route -n|awk '{ if ($8 ==\"eth0\" && $2 != \"0.0.0.0\") print \"route del default gw \" $2; }'`"
    vm.provision :shell, run: "always", inline: "sed -i 's/^#PermitRootLogin/PermitRootLogin/' /etc/ssh/sshd_config"
    vm.provision :shell, run: "always", inline: "sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config && systemctl restart sshd"
    #vm.provision :ansible do |ansible|
    #    ansible.host_key_checking = false
    #    ansible.playbook = "ansible/playbook.yml"
    #    ansible.verbose = "vv"
    #    #ansible.extra_vars = {}
    #end
    #vm.provision :shell, :inline => "cd /opt/devstack; sudo -u ubuntu env HOME=/home/ubuntu ./stack.sh"
    # interface should match external_interface var in devstack.yml
    #vm.provision :shell, :inline => "ovs-vsctl add-port br-ex enp0s9"
    #vm.provision :shell, :inline => "virsh net-destroy default"
end
#  --------------  Provision End ----------------------------------------------------

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    if Vagrant.has_plugin?("vagrant-cachier")
        config.cache.scope = :box
    end

    if Vagrant.has_plugin?("vagrant-proxyconf") && conf['proxy']
        config.proxy.http     = conf['proxy']
        config.proxy.https    = conf['proxy']
        config.proxy.no_proxy = "localhost,127.0.0.1"
    end

    # resolve "stdin: is not a tty warning", related issue and proposed 
    # fix: https://github.com/mitchellh/vagrant/issues/1673
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"
    config.ssh.forward_agent = true

    if conf['provider'] == 'virtualbox'
        config_vb_network(config.vm, conf)
        config_vb_provider(config.vm, conf)
    end

    if conf['provider'] == 'libvirt'
        config_lv_provider(config.vm, conf)
        config_lv_define_box1(config.vm, conf)
    end

    if conf['local_sync_folder']
        config.vm.synced_folder conf['local_sync_folder'], "/vagrant"
    end
end

