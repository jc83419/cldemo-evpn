# Created by Topology-Converter v4.6.1_dev
#    https://github.com/cumulusnetworks/topology_converter
#    using topology data from: topology_kvm.dot
#
#    NOTE: in order to use this Vagrantfile you will need:
#       -Vagrant(v1.8.1+) installed: http://www.vagrantup.com/downloads
#       -Cumulus Plugin for Vagrant installed: $ vagrant plugin install vagrant-cumulus
#       -the "helper_scripts" directory that comes packaged with topology-converter.py
#       -Virtualbox installed: https://www.virtualbox.org/wiki/Downloads



# Check required plugins
REQUIRED_PLUGINS = %w(vagrant-cumulus)
exit unless REQUIRED_PLUGINS.all? do |plugin|
  Vagrant.has_plugin?(plugin) || (
    puts "The #{plugin} plugin is required. Please install it with:"
    puts "$ vagrant plugin install #{plugin}"
    false
  )
end


$script = <<-SCRIPT
if grep -q -i 'cumulus' /etc/lsb-release &> /dev/null; then
    echo "### RUNNING CUMULUS EXTRA CONFIG ###"
    source /etc/lsb-release
    if [[ $DISTRIB_RELEASE =~ ^2.* ]]; then
        echo "  INFO: Detected a 2.5.x Based Release"

        echo "  adding fake cl-acltool..."
        echo -e "#!/bin/bash\nexit 0" > /usr/bin/cl-acltool
        chmod 755 /usr/bin/cl-acltool

        echo "  adding fake cl-license..."
        echo -e "#!/bin/bash\nexit 0" > /usr/bin/cl-license
        chmod 755 /usr/bin/cl-license

        echo "  Disabling default remap on Cumulus VX..."
        mv -v /etc/init.d/rename_eth_swp /etc/init.d/rename_eth_swp.backup

        echo "### Rebooting to Apply Remap..."
        reboot

    elif [[ $DISTRIB_RELEASE =~ ^3.* ]]; then
        echo "  INFO: Detected a 3.x Based Release"
        echo "  Disabling default remap on Cumulus VX..."
        mv -v /etc/hw_init.d/S10rename_eth_swp.sh /etc/S10rename_eth_swp.sh.backup
        #echo "### Applying Remap without Reboot..."
        #~/apply_udev.py --apply
        #echo "### Performing IFRELOAD to Apply any Latent Interface Config..."
        #ifreload -a 2>&1
        echo "### Disabling ZTP service..."
        systemctl stop ztp.service
        ztp -d 2>&1
        echo "### Resetting ZTP to work next boot..."
        ztp -R 2>&1
        echo "### Rebooting Switch to Apply Remap..."
        reboot
    fi
    echo "### DONE ###"
else
    echo "### Rebooting to Apply Remap..."
    reboot
fi
SCRIPT

Vagrant.configure("2") do |config|
  wbid = 1490993080

  config.vm.provider "virtualbox" do |v|
    v.gui=false

  end




  ##### DEFINE VM for oob-mgmt-server #####
  config.vm.define "oob-mgmt-server" do |device|
    device.vm.hostname = "oob-mgmt-server"
    device.vm.box = "yk0/ubuntu-xenial"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_oob-mgmt-server"
      v.memory = 1024
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth1 --> oob-mgmt-switch:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net52", auto_config: false , :mac => "443839000059"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Shorten Boot Process - Applies to Ubuntu Only - remove \"Wait for Network\"
    device.vm.provision :shell , inline: "sed -i 's/sleep [0-9]*/sleep 1/' /etc/init/failsafe.conf 2>/dev/null || true"


    #Copy over DHCP files and MGMT Network Files
    device.vm.provision "file", source: "./helper_scripts/dhcpd.conf", destination: "~/dhcpd.conf"
    device.vm.provision "file", source: "./helper_scripts/dhcpd.hosts", destination: "~/dhcpd.hosts"
    device.vm.provision "file", source: "./helper_scripts/hosts", destination: "~/hosts"
    device.vm.provision "file", source: "./helper_scripts/ansible_hostfile", destination: "~/ansible_hostfile"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_oob_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:59 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:59", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for oob-mgmt-switch #####
  config.vm.define "oob-mgmt-switch" do |device|
    device.vm.hostname = "oob-mgmt-switch"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_oob-mgmt-switch"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> NOTHING:NOTHING
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net58", auto_config: false , :mac => "443839000063"

      # link for swp1 --> oob-mgmt-server:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net52", auto_config: false , :mac => "44383900005a"

      # link for swp2 --> server01:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net41", auto_config: false , :mac => "443839000046"

      # link for swp4 --> server03:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net16", auto_config: false , :mac => "44383900001d"

      # link for swp6 --> leaf01:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net17", auto_config: false , :mac => "44383900001e"

      # link for swp7 --> leaf02:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net37", auto_config: false , :mac => "443839000040"

      # link for swp8 --> leaf03:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net25", auto_config: false , :mac => "44383900002d"

      # link for swp9 --> leaf04:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net32", auto_config: false , :mac => "443839000038"

      # link for swp10 --> spine01:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net28", auto_config: false , :mac => "443839000032"

      # link for swp11 --> spine02:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net10", auto_config: false , :mac => "443839000012"

      # link for swp12 --> exit01:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net7", auto_config: false , :mac => "44383900000d"

      # link for swp13 --> exit02:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net46", auto_config: false , :mac => "44383900004f"

      # link for swp14 --> site02:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net39", auto_config: false , :mac => "443839000043"

      # link for swp15 --> internet:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net33", auto_config: false , :mac => "443839000039"

      # link for swp16 --> isp:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net50", auto_config: false , :mac => "443839000056"

      # link for swp17 --> server05:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net29", auto_config: false , :mac => "443839000033"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc13', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc14', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc15', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc16', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc17', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_oob_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:63 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:63", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:5a --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:5a", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:46 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:46", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1d --> swp4"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1d", NAME="swp4", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1e --> swp6"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1e", NAME="swp6", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:40 --> swp7"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:40", NAME="swp7", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2d --> swp8"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2d", NAME="swp8", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:38 --> swp9"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:38", NAME="swp9", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:32 --> swp10"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:32", NAME="swp10", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:12 --> swp11"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:12", NAME="swp11", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0d --> swp12"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0d", NAME="swp12", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4f --> swp13"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4f", NAME="swp13", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:43 --> swp14"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:43", NAME="swp14", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:39 --> swp15"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:39", NAME="swp15", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:56 --> swp16"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:56", NAME="swp16", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:33 --> swp17"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:33", NAME="swp17", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for exit02 #####
  config.vm.define "exit02" do |device|
    device.vm.hostname = "exit02"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_exit02"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp13
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net46", auto_config: false , :mac => "a00000000042"

      # link for swp44 --> internet:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net38", auto_config: false , :mac => "443839000042"

      # link for swp45 --> exit02:swp46
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net27", auto_config: false , :mac => "443839000030"

      # link for swp46 --> exit02:swp45
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net27", auto_config: false , :mac => "443839000031"

      # link for swp47 --> exit02:swp48
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net31", auto_config: false , :mac => "443839000036"

      # link for swp48 --> exit02:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net31", auto_config: false , :mac => "443839000037"

      # link for swp49 --> exit01:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net21", auto_config: false , :mac => "443839000026"

      # link for swp50 --> exit01:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net11", auto_config: false , :mac => "443839000014"

      # link for swp51 --> spine01:swp29
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net18", auto_config: false , :mac => "44383900001f"

      # link for swp52 --> spine02:swp29
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net51", auto_config: false , :mac => "443839000057"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/exit02/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/exit02/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/exit02/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:42 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:42", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:42 --> swp44"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:42", NAME="swp44", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:30 --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:30", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:31 --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:31", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:36 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:36", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:37 --> swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:37", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:26 --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:26", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:14 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:14", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1f --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1f", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:57 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:57", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for exit01 #####
  config.vm.define "exit01" do |device|
    device.vm.hostname = "exit01"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_exit01"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp12
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net7", auto_config: false , :mac => "a00000000041"

      # link for swp44 --> internet:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net4", auto_config: false , :mac => "443839000008"

      # link for swp45 --> exit01:swp46
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net42", auto_config: false , :mac => "443839000047"

      # link for swp46 --> exit01:swp45
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net42", auto_config: false , :mac => "443839000048"

      # link for swp47 --> exit01:swp48
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net9", auto_config: false , :mac => "443839000010"

      # link for swp48 --> exit01:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net9", auto_config: false , :mac => "443839000011"

      # link for swp49 --> exit02:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net21", auto_config: false , :mac => "443839000025"

      # link for swp50 --> exit02:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net11", auto_config: false , :mac => "443839000013"

      # link for swp51 --> spine01:swp30
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net5", auto_config: false , :mac => "443839000009"

      # link for swp52 --> spine02:swp30
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net54", auto_config: false , :mac => "44383900005d"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/exit01/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/exit01/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/exit01/Quagga.conf", destination: "~/Quagga.conf"

    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:41 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:41", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:08 --> swp44"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:08", NAME="swp44", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:47 --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:47", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:48 --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:48", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:10 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:10", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:11 --> swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:11", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:25 --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:25", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:13 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:13", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:09 --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:09", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:5d --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:5d", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for isp #####
  config.vm.define "isp" do |device|
    device.vm.hostname = "isp"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_isp"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp16
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net50", auto_config: false , :mac => "a00000000052"

      # link for swp1 --> site02:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net45", auto_config: false , :mac => "44383900004e"

      # link for swp47 --> internet:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net24", auto_config: false , :mac => "44383900002b"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/isp/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/isp/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/isp/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:52 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:52", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4e --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4e", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2b --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2b", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for spine02 #####
  config.vm.define "spine02" do |device|
    device.vm.hostname = "spine02"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_spine02"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp11
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net10", auto_config: false , :mac => "a00000000022"

      # link for swp1 --> leaf01:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net20", auto_config: false , :mac => "443839000024"

      # link for swp2 --> leaf02:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net56", auto_config: false , :mac => "443839000062"

      # link for swp3 --> leaf03:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net14", auto_config: false , :mac => "44383900001a"

      # link for swp4 --> leaf04:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net43", auto_config: false , :mac => "44383900004a"

      # link for swp29 --> exit02:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net51", auto_config: false , :mac => "443839000058"

      # link for swp30 --> exit01:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net54", auto_config: false , :mac => "44383900005e"

      # link for swp31 --> spine01:swp31
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net44", auto_config: false , :mac => "44383900004c"

      # link for swp32 --> spine01:swp32
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net35", auto_config: false , :mac => "44383900003d"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/spine02/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/spine02/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/spine02/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:22 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:22", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:24 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:24", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:62 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:62", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1a --> swp3"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1a", NAME="swp3", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4a --> swp4"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4a", NAME="swp4", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:58 --> swp29"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:58", NAME="swp29", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:5e --> swp30"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:5e", NAME="swp30", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4c --> swp31"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4c", NAME="swp31", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3d --> swp32"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3d", NAME="swp32", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for spine01 #####
  config.vm.define "spine01" do |device|
    device.vm.hostname = "spine01"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_spine01"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp10
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net28", auto_config: false , :mac => "a00000000021"

      # link for swp1 --> leaf01:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net49", auto_config: false , :mac => "443839000055"

      # link for swp2 --> leaf02:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net22", auto_config: false , :mac => "443839000028"

      # link for swp3 --> leaf03:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net47", auto_config: false , :mac => "443839000051"

      # link for swp4 --> leaf04:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net36", auto_config: false , :mac => "44383900003f"

      # link for swp29 --> exit02:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net18", auto_config: false , :mac => "443839000020"

      # link for swp30 --> exit01:swp51
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net5", auto_config: false , :mac => "44383900000a"

      # link for swp31 --> spine02:swp31
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net44", auto_config: false , :mac => "44383900004b"

      # link for swp32 --> spine02:swp32
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net35", auto_config: false , :mac => "44383900003c"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/spine01/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/spine01/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/spine01/Quagga.conf", destination: "~/Quagga.conf"

    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:21 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:21", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:55 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:55", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:28 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:28", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:51 --> swp3"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:51", NAME="swp3", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3f --> swp4"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3f", NAME="swp4", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:20 --> swp29"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:20", NAME="swp29", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0a --> swp30"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0a", NAME="swp30", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4b --> swp31"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4b", NAME="swp31", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3c --> swp32"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3c", NAME="swp32", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf04 #####
  config.vm.define "leaf04" do |device|
    device.vm.hostname = "leaf04"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_leaf04"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp9
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net32", auto_config: false , :mac => "a00000000014"

      # link for swp1 --> server03:eth2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net55", auto_config: false , :mac => "443839000060"

      # link for swp45 --> leaf04:swp46
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net13", auto_config: false , :mac => "443839000017"

      # link for swp46 --> leaf04:swp45
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net13", auto_config: false , :mac => "443839000018"

      # link for swp47 --> leaf04:swp48
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net30", auto_config: false , :mac => "443839000034"

      # link for swp48 --> leaf04:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net30", auto_config: false , :mac => "443839000035"

      # link for swp49 --> leaf03:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net26", auto_config: false , :mac => "44383900002f"

      # link for swp50 --> leaf03:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net3", auto_config: false , :mac => "443839000006"

      # link for swp51 --> spine01:swp4
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net36", auto_config: false , :mac => "44383900003e"

      # link for swp52 --> spine02:swp4
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net43", auto_config: false , :mac => "443839000049"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/leaf04/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/leaf04/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/leaf04/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:14 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:14", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:60 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:60", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:17 --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:17", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:18 --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:18", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:34 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:34", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:35 --> swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:35", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2f --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2f", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:06 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:06", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3e --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3e", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:49 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:49", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf02 #####
  config.vm.define "leaf02" do |device|
    device.vm.hostname = "leaf02"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_leaf02"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp7
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net37", auto_config: false , :mac => "a00000000012"

      # link for swp1 --> server01:eth2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net12", auto_config: false , :mac => "443839000016"

      # link for swp45 --> leaf02:swp46
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net6", auto_config: false , :mac => "44383900000b"

      # link for swp46 --> leaf02:swp45
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net6", auto_config: false , :mac => "44383900000c"

      # link for swp47 --> leaf02:swp48
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net53", auto_config: false , :mac => "44383900005b"

      # link for swp48 --> leaf02:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net53", auto_config: false , :mac => "44383900005c"

      # link for swp49 --> leaf01:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net8", auto_config: false , :mac => "44383900000f"

      # link for swp50 --> leaf01:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net1", auto_config: false , :mac => "443839000002"

      # link for swp51 --> spine01:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net22", auto_config: false , :mac => "443839000027"

      # link for swp52 --> spine02:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net56", auto_config: false , :mac => "443839000061"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/leaf02/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/leaf02/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/leaf02/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:12 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:12", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:16 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:16", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0b --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0b", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0c --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0c", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:5b --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:5b", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:5c --> swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:5c", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0f --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0f", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:02 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:02", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:27 --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:27", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:61 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:61", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf03 #####
  config.vm.define "leaf03" do |device|
    device.vm.hostname = "leaf03"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_leaf03"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp8
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net25", auto_config: false , :mac => "a00000000013"

      # link for swp1 --> server03:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net19", auto_config: false , :mac => "443839000022"

      # link for swp45 --> leaf03:swp46
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net23", auto_config: false , :mac => "443839000029"

      # link for swp46 --> leaf03:swp45
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net23", auto_config: false , :mac => "44383900002a"

      # link for swp47 --> leaf03:swp48
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net48", auto_config: false , :mac => "443839000052"

      # link for swp48 --> leaf03:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net48", auto_config: false , :mac => "443839000053"

      # link for swp49 --> leaf04:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net26", auto_config: false , :mac => "44383900002e"

      # link for swp50 --> leaf04:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net3", auto_config: false , :mac => "443839000005"

      # link for swp51 --> spine01:swp3
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net47", auto_config: false , :mac => "443839000050"

      # link for swp52 --> spine02:swp3
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net14", auto_config: false , :mac => "443839000019"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/leaf03/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/leaf03/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/leaf03/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:13 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:13", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:22 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:22", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:29 --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:29", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2a --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2a", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:52 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:52", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:53 --> swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:53", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2e --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2e", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:05 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:05", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:50 --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:50", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:19 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:19", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf01 #####
  config.vm.define "leaf01" do |device|
    device.vm.hostname = "leaf01"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_leaf01"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp6
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net17", auto_config: false , :mac => "a00000000011"

      # link for swp1 --> server01:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net2", auto_config: false , :mac => "443839000004"

      # link for swp45 --> leaf01:swp46
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net15", auto_config: false , :mac => "44383900001b"

      # link for swp46 --> leaf01:swp45
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net15", auto_config: false , :mac => "44383900001c"

      # link for swp47 --> leaf01:swp48
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net40", auto_config: false , :mac => "443839000044"

      # link for swp48 --> leaf01:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net40", auto_config: false , :mac => "443839000045"

      # link for swp49 --> leaf02:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net8", auto_config: false , :mac => "44383900000e"

      # link for swp50 --> leaf02:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net1", auto_config: false , :mac => "443839000001"

      # link for swp51 --> spine01:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net49", auto_config: false , :mac => "443839000054"

      # link for swp52 --> spine02:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net20", auto_config: false , :mac => "443839000023"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/leaf01/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/leaf01/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/leaf01/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:11 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:11", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:04 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:04", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1b --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1b", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1c --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1c", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:44 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:44", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:45 --> swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:45", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0e --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0e", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:01 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:01", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:54 --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:54", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:23 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:23", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for site02 #####
  config.vm.define "site02" do |device|
    device.vm.hostname = "site02"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_site02"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp14
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net39", auto_config: false , :mac => "a00000000051"

      # link for swp1 --> server05:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net34", auto_config: false , :mac => "44383900003b"

      # link for swp51 --> isp:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net45", auto_config: false , :mac => "44383900004d"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/site02/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/site02/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/site02/Quagga.conf", destination: "~/Quagga.conf"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:51 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:51", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3b --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3b", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:4d --> swp51"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:4d", NAME="swp51", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for server01 #####
  config.vm.define "server01" do |device|
    device.vm.hostname = "server01"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_server01"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net41", auto_config: false , :mac => "a00000000031"

      # link for eth1 --> leaf01:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net2", auto_config: false , :mac => "443839000003"

      # link for eth2 --> leaf02:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net12", auto_config: false , :mac => "443839000015"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/server01/interfaces", destination: "~/interfaces"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:31 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:31", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:03 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:03", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:15 --> eth2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:15", NAME="eth2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for server03 #####
  config.vm.define "server03" do |device|
    device.vm.hostname = "server03"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_server03"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp4
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net16", auto_config: false , :mac => "a00000000033"

      # link for eth1 --> leaf03:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net19", auto_config: false , :mac => "443839000021"

      # link for eth2 --> leaf04:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net55", auto_config: false , :mac => "44383900005f"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    device.vm.provision "file", source: "./config/server03/interfaces", destination: "~/interfaces"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:33 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:33", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:21 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:21", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:5f --> eth2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:5f", NAME="eth2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for server05 #####
  config.vm.define "server05" do |device|
    device.vm.hostname = "server05"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_server05"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp17
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net29", auto_config: false , :mac => "a00000000054"

      # link for eth1 --> site02:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net34", auto_config: false , :mac => "44383900003a"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"

    device.vm.provision "file", source: "./config/server05/interfaces", destination: "~/interfaces"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_server.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:54 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:54", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3a --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3a", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for internet #####
  config.vm.define "internet" do |device|
    device.vm.hostname = "internet"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.box_version = "3.2.1"
    device.vm.provider "virtualbox" do |v|
      v.name = "1490993080_internet"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp15
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net33", auto_config: false , :mac => "a00000000053"

      # link for swp1 --> exit01:swp44
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net4", auto_config: false , :mac => "443839000007"

      # link for swp2 --> exit02:swp44
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net38", auto_config: false , :mac => "443839000041"

      # link for swp47 --> isp:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net24", auto_config: false , :mac => "44383900002c"


    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile  2>/dev/null && echo 'Ignore the previous error, fixing this now...') || true;"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

    # Copy over configuration files
    device.vm.provision "file", source: "./config/internet/interfaces", destination: "~/interfaces"
    device.vm.provision "file", source: "./config/internet/daemons", destination: "~/daemons"
    device.vm.provision "file", source: "./config/internet/Quagga.conf", destination: "~/Quagga.conf"

    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: a0:00:00:00:00:53 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="a0:00:00:00:00:53", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:07 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:07", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:41 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:41", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2c --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2c", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule

      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = swp48"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="swp48", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end



end
