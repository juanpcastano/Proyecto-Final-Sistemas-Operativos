# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define :vm1 do |vm1|
    vm1.vm.box = "ubuntu/bionic64"

    vm1.vm.network "private_network", ip: "192.168.56.21"

    vm1.vm.synced_folder "./html", "/var/www/html"

    vm1.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end

    #vm1.ssh.forward_agent = true
    #public_key = File.read(File.expand_path("ssh/new_vagrant_key.pub"))

    vm1.vm.provision "shell", inline: <<-SHELL
      sudo useradd -m -g admin -s /bin/bash admin 
      echo "admin:password_admin" | sudo chpasswd

      sudo useradd -m -s /bin/bash tester 
      echo "tester:password_tester" | sudo chpasswd

      sudo useradd -m -s /bin/bash appUser 
      echo "appUser:password_appUser" | sudo chpasswd

      for user in admin tester appUser; do
          sudo mkdir -p /home/$user/.ssh
          sudo chown -R $user:$user /home/$user/.ssh
          sudo chmod 700 /home/$user/.ssh
      done
      ssh-keygen -t rsa -N "" -f /tmp/host_ssh_key

      for user in admin tester appUser; do
          sudo cp /tmp/host_ssh_key.pub /home/$user/.ssh/authorized_keys
          sudo chown $user:$user /home/$user/.ssh/authorized_keys
          sudo chmod 600 /home/$user/.ssh/authorized_keys
      done

      echo "admin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/admin

      sudo cp /tmp/host_ssh_key /vagrant/host_ssh_key
      rm /tmp/host_ssh_key /tmp/host_ssh_key.pub

    SHELL

  end
  config.vm.define :vm2 do |vm2|
    vm2.vm.box = "ubuntu/bionic64"

    vm2.vm.network "private_network", ip: "192.168.56.22"

    vm2.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end
  end
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.inventory_path = "inventory.ini"
  end
end
