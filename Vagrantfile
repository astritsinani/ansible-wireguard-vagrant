
Vagrant.configure("2") do |config|
  config.vm.box = "utm/bookworm"

  nodes = {
    "router" => "192.168.56.10",
    "client1" => "192.168.56.11",
    "client2" => "192.168.56.12"
  }

  nodes.each do |name, ip|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip

      node.vm.provider "virtualbox" do |vb|
        vb.cpus = 1
        vb.memory = 512
      end

      node.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/site.yml"
        ansible.inventory_path = "ansible/inventory.ini"
        ansible.become = true
      end
    end
  end
end
