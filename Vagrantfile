Vagrant.configure("2") do |config|
config.vm.box = "ubuntu/jammy64"
public_key = File.expand_path("~/.ssh/id_ed25519.pub")
(1..5).each do |i|
config.vm.define "node#{i}" do |node|
node.vm.hostname = "node#{i}"

  # Bridged Network
  node.vm.network "public_network", bridge: "eno2"
  # Copy host public key to vagrant user's authorized_keys 
  node.vm.provision "shell", inline: <<-SHELL 
    mkdir -p /home/vagrant/.ssh 
    chmod 700 /home/vagrant/.ssh 
    echo "#{File.read(public_key).strip}" >> /home/vagrant/.ssh/authorized_keys 
    chmod 600 /home/vagrant/.ssh/authorized_keys 
    chown -R vagrant:vagrant /home/vagrant/.ssh 
  SHELL

  node.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
    vb.memory = 4096
    vb.name = "postgres-node#{i}"
  end
end

end
end

