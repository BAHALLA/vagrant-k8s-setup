# -*- mode: ruby -*- 
# vi: set ft=ruby : 

BOX_IMAGE = "rockylinux/9"
NODE_COUNT = 2 

Vagrant.configure("2") do |config|  

  config.vm.define "console" do |subconfig|  
    subconfig.vm.box = BOX_IMAGE   
    subconfig.vm.hostname = "console"   
    subconfig.vm.network :private_network, ip: "10.0.0.9"  
    subconfig.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "2048"]
    end   
  end

  config.vm.define "master" do |subconfig|  
      subconfig.vm.box = BOX_IMAGE   
      subconfig.vm.hostname = "master"   
      subconfig.vm.network :private_network, ip: "10.0.0.10"  
      subconfig.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "2048"]
      end
  end     
  (1..NODE_COUNT).each do |i|   
     config.vm.define "node#{i}" do |subconfig|  
          subconfig.vm.box = BOX_IMAGE      
          subconfig.vm.hostname = "node#{i}"   
          subconfig.vm.network :private_network, ip: "10.0.0.#{i + 10}" 
          subconfig.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "3072"]
            vb.customize ["modifyvm", :id, "--cpus", "2"]
          end
     end   
   end   
end
