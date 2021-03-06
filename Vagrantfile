# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

	config.vm.box = 'centos/7'
	config.vm.box_version = '=1703.01'
	
	# the croit controller VM
	config.vm.define :croit, primary: true do |config|
		config.vm.network "forwarded_port", guest: 8080, host: 8080
		config.vm.network "forwarded_port", guest: 443, host: 8443
		config.vm.network "private_network", ip: "192.168.0.2", libvirt__network_name: "croit_pxe", :libvirt__dhcp_enabled => false, virtualbox__intnet: "croit_pxe"
		config.vm.provider "virtualbox" do |vb|
			vb.memory = '1536'
			vb.cpus = '4' # makes the initial setup much faster
		end
		config.vm.provision "shell", privileged: true, inline: <<-SHELL
			# this seems to be necessary sometimes
			yum makecache
			yum install -y yum-utils centos-release-ceph-jewel
			yum install -y ceph
			yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
			yum makecache fast
			yum -y install docker-ce
			systemctl start docker
			docker login -u croit -p beta registry.croit.io/v2
			echo "Downloading and starting Docker image (1.5 GB)."
			echo "This will take several minutes..."
			docker run --net=host --name croit -d registry.croit.io/v2/croit:latest
		SHELL
		config.vm.provision "shell", privileged: true, run: "always", inline: <<-SHELL
			systemctl start docker
			docker login -u croit -p beta registry.croit.io/v2
			echo "Trying to update Docker image, this might take a few minutes"
			docker pull registry.croit.io/v2/croit:latest
			docker start croit
		SHELL
	end

	# 5 identical test VMs
	(1..5).each do |i|
		config.vm.define :"ceph#{i}", autostart: false do |config|
			config.vm.box = 'c33s/empty'
			config.vm.box_version = '=0.1.0'
			config.vm.provider :virtualbox do |vb|
				vb.gui = 'true'
				vb.memory = '2048'
				vb.cpus = '2'
				vb.customize [
					'modifyvm', :id,
					# the image enables usb2.0 by default which requires the oracle extensions
					# we don't want that dependency
					'--usbehci', 'off',
					# virtualbox complains otherwise
					'--vram', '16'
				]
				# network setup
				vb.customize [
					'modifyvm', :id,
					'--nic1', 'intnet',
					'--intnet1', 'croit_pxe',
					# make sure it only boots via network
					'--boot1', 'net', '--boot2', 'none', '--boot3', 'none', '--boot4', 'none',
				]
				# no, we don't want an annoying prompt
				vb.customize [ 'setextradata', :id, 'GUI/FirstRun', 'no' ]
			end
			# running the disk commands multiple times fails
			# and there is no way to ignore failures in vb.customize
			unless created?("ceph#{i}")
				config.vm.provider :virtualbox do |vb|
					# we need to change the MAC on the first startup (but only on the first, otherwise it will be re-detected)
					# yes, it just re-uses the same mac by default...
					vb.customize [
						'modifyvm', :id,
						'--macaddress1', 'auto'
					]
					# delete the cd drive
					vb.customize [
						'storageattach', :id,
						'--storagectl', 'IDE',
						'--medium', 'none',
						'--port', 1,
						'--device', 0
					]
					# delete the default (empty) disk
					vb.customize [
						'storageattach', :id,
						'--storagectl', 'SATA',
						'--medium', 'none',
						'--port', 0,
						'--device', 0
					]
					# create our disks, suggested usage:
					# disk 1: mon (1 GB)
					# disk 2: journal (1 GB)
					# disk 3: osd (8 GB)
					add_disk(vb, "./ceph#{i}-disk1.vdi", 0, 1024, 'on')
					add_disk(vb, "./ceph#{i}-disk2.vdi", 1, 1024, 'on')
					add_disk(vb, "./ceph#{i}-disk3.vdi", 2, 8192, 'off')
				end
			end
		end
	end
end

def add_disk(vb, name, port, size, ssd)
	unless File.exist?(name)
		vb.customize [
			'createmedium', 'disk',
			'--filename', name,
			'--size', size
		]
	end
	vb.customize [
		'storageattach', :id,
		'--storagectl', 'SATA',
		'--type', 'hdd',
		'--medium', name,
		'--port', port,
		'--nonrotational', ssd
	]
end

# Check if a VM was already created before
def created?(vm_name, provider='virtualbox')
	File.exist?(".vagrant/machines/#{vm_name}/#{provider}/id")
end

