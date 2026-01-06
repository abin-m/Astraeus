Vagrant.configure("2") do |config|
    config.vm.box = "hashicorp-education/ubuntu-24-04"
    config.vm.box_version = "0.1.0"
  
    # Web Server
    config.vm.define "web" do |web|
      web.vm.network "private_network", ip: "192.168.56.10"
      web.vm.hostname = "webserver"
    end
  
    # Database Server
    config.vm.define "db" do |db|
      db.vm.network "private_network", ip: "192.168.56.11"
      db.vm.hostname = "dbserver" 
    end
  end