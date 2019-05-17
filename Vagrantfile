# -*- mode: ruby -*-
# vi: set ft=ruby :

$parted_sdb = <<-SCRIPT
parted --script /dev/sdb \
unit % \
mklabel msdos \
mkpart primary ext3 1 100% \
quit
SCRIPT

Vagrant.configure("2") do |config|

  config.vm.box = "centos/7"

  config.ssh.insert_key = false
  config.vm.provision "file", source: "keys/config",  destination: "~/.ssh/config"
  config.vm.provision "file", source: "keys/vagrant", destination: "~/.ssh/id_rsa"
  config.vm.provision "file", source: "keys/vagrant.pub", destination: "~/.ssh/id_rsa.pub"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa.pub"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/config"

  # Define the first server. This will be the master of pcs cluster.
  config.vm.define "rear01", primary: true do |rear01|
    rear01.vm.hostname = "rear01"
    rear01.vm.network "private_network", ip: "192.168.50.10"
    rear01.vm.provider :virtualbox do |vb|
        vb.memory = 1024
        vb.cpus = 2
        #
        # Add a second disk that will exported via NFS (ReaR)
        # Only creates the file if it doesn't exists already.
        file_to_disk = File.realpath(".").to_s + "/rear01b.vdi"
        if !File.exist?(file_to_disk)
            vb.customize [
                'createhd',
                '--filename', file_to_disk,
                '--format', 'VDI',
                '--size', 15 * 1024 # 15GB
            ]
            vb.customize [
                'storagectl', :id,
                '--name', 'rearctl',
                '--add', 'sata'
            ]
            vb.customize [
                'storageattach', :id,
                '--storagectl', 'rearctl',
                '--port', 0,
                '--type', 'hdd', '--medium',
                file_to_disk
            ]
        end
    end
    rear01.vm.provision "shell", inline: $parted_sdb
  end

  # Define the second server. This will be the slave of pcs cluster.
  config.vm.define "rear02" do |rear02|
    rear02.vm.hostname = "rear02"
    rear02.vm.network "private_network", ip: "192.168.50.11"
    rear02.vm.provider :virtualbox do |vb|
        vb.memory = 2048
        vb.cpus = 2
    end
  end

end
