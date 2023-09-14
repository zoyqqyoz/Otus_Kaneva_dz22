#ruby 
# -*- mode: ruby -*- 
# vi: set ft=ruby : vsa
Vagrant.configure ("2") do |config| 
 config.vm.box = "centos/8"
 config.vm.synced_folder ".", "/vagrant", disabled: true
 config.vm.provider "virtualbox" do |v| 
 v.memory = 512 
 v.cpus = 1 
 end

 config.vm.define "server" do |server| 
 server.vm.network "private_network", ip: "192.168.56.10",  virtualbox__intnet: "net1" 
 server.vm.hostname = "server"
 server.vm.provision:file, source: '~/conf/', destination: "/tmp"
  server.vm.provision "shell", inline: <<-SHELL
    mkdir -p ~root/.ssh
    cp ~vagrant/.ssh/auth* ~root/.ssh
    cd /etc/yum.repos.d/
    sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
    sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
    sudo yum install -y epel-release; sudo yum install -y openvpn easy-rsa
    sudo setenforce 0
    sudo sed -i 's/=enforcing/=disabled/g' /etc/selinux/config
    cd /etc/openvpn/
    sudo /usr/share/easy-rsa/3.0.8/easyrsa init-pki
    sudo echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass
    sudo echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass
    sudo echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server
    sudo /usr/share/easy-rsa/3.0.8/easyrsa gen-dh
    sudo openvpn --genkey --secret ca.key
    sudo echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass
    sudo echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client
    sudo cp -r /tmp/server.conf /etc/openvpn/; cp /tmp/openvpn@.service /etc/systemd/system/
    sudo echo 'iroute 10.10.10.0 255.255.255.0' > /etc/openvpn/client/client
    sudo systemctl enable openvpn@server && sudo systemctl start openvpn@server
    SHELL
  end

end
