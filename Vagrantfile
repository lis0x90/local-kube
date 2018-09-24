# -*- mode: ruby -*-
# vi: set ft=ruby :

IP_DNS = "192.168.1.254"
IP_MASTER1 = "192.168.1.10"

NODE_COUNT = 1
IP_NODE_TEMPL = "192.168.1.%s"
KUBER_VERSION="1.11.3"


# TODO 
# 
# sudo vim /etc/sysconfig/kubelet
# add at the end of DAEMON_ARGS string: --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice
# sudo systemctl restart kubelet

# TODO
# on master 
# edit: /etc/kubernetes/manifests/kube-apiserver.yaml 
# --enable-admission-plugins=NodeRestriction,ValidatingAdmissionWebhook,MutatingAdmissionWebhook


nodeInit = <<-SHELL
			set -ex
			sudo yum -y install yum-utils vim bash-compl* mc wget nettools bind-utils device-mapper-persistent-data lvm2
			
			sudo setenforce 0
			sudo sed -i 's/SELINUX=.\\+/SELINUX=disabled/g' /etc/selinux/config

			sudo swapoff -a
			sudo sed -i 's@/.\\+swap@# \\0@' /etc/fstab

			sudo modprobe br_netfilter			
			sudo echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
			sudo sysctl -p

			sudo modprobe ip_vs
			sudo modprobe ip_vs_wrr 
			sudo modprobe ip_vs_sh 
			sudo modprobe ip_vs_rr
			sudo echo ip_vs >/etc/modules-load.d/ip_vs.conf
			sudo echo ip_vs_wrr >/etc/modules-load.d/ip_vs_wrr.conf
			sudo echo ip_vs_sh >/etc/modules-load.d/ip_vs_sh.conf
			sudo echo ip_vs_rr >/etc/modules-load.d/ip_vs_rr.conf
			
			sudo mkdir -p /etc/docker/certs.d/docker-registry:5000 
			sudo cp -v /vagrant/registry/certs/domain.crt /etc/docker/certs.d/docker-registry:5000/ca.crt
			sudo cp -v /vagrant/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/docker-registry.crt
			sudo update-ca-trust enable
			sudo update-ca-trust extract

			sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
			sudo yum install -y docker kubelet-#{KUBER_VERSION} kubeadm-#{KUBER_VERSION} kubectl-#{KUBER_VERSION} kubernetes-cni-#{KUBER_VERSION}
			sudo systemctl enable docker.service
			sudo systemctl enable kubelet

			sudo echo "nameserver #{IP_DNS}" >/etc/resolv.conf
			# lock /etc/resolv.conf from modifications by dhcp script 
			sudo chattr +i /etc/resolv.conf 
			
			kubectl completion bash >>/home/vagrant/.bashrc
SHELL


Vagrant.configure("2") do |config|
	
	# this vm for two roles: dns and docker-registry
	config.vm.define "dns" do |cfg|
		cfg.vm.box = "bento/centos-7.4"
		cfg.vm.network "private_network", ip: IP_DNS
		cfg.vm.hostname = "dns"
		cfg.vm.provider :virtualbox do |vbox|
			vbox.memory = "1024"
		end
		
 		nodesDns = ""
		for n in 1..NODE_COUNT do
			ip = IP_NODE_TEMPL % (20 + n) 
			nodesDns += "sudo echo \"#{ip} node#{n}\" >> /etc/hosts\n"
		end
		cfg.vm.provision "shell", inline: <<-SHELL
		  	set -ex
			
			sudo yum install -y vim bash-compl* mc wget dnsmasq bind-utils docker 
			sudo echo "#{IP_DNS} dns" >> /etc/hosts
			sudo echo "#{IP_DNS} docker-registry" >> /etc/hosts
			sudo echo "#{IP_MASTER1} master1" >> /etc/hosts
			#{nodesDns}
			
			sudo systemctl start dnsmasq.service

			# start private docker registry
			sudo systemctl start docker 
			sudo mkdir -p /vagrant/registry/{storage,certs}
			openssl req -newkey rsa:4096 -nodes -sha256 -keyout /vagrant/registry/certs/domain.key -x509 -days 365 -out /vagrant/registry/certs/domain.crt -subj "/C=RU/ST=Denial/L=Saint-Petersburg/O=NetCracker/CN=docker-registry"
			# trust self-signed cert for docker-registry
			sudo mkdir -p /etc/docker/certs.d/docker-registry:5000 
			sudo cp -v /vagrant/registry/certs/domain.crt /etc/docker/certs.d/docker-registry:5000/ca.crt
			sudo cp -v /vagrant/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/docker-registry.crt
			
	
			sudo docker run -d -p #{IP_DNS}:5000:5000 -p #{IP_DNS}:443:443 \
				--restart=always \
				--name registry \
				-v /vagrant/registry/storage:/var/lib/registry \
				-v /vagrant/registry/certs:/certs \
				-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
				-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
				-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
				registry:2
		SHELL
	end

	config.vm.define "master1" do |cfg|	
		cfg.vm.box = "bento/centos-7.4"
		cfg.vm.network "private_network", ip: IP_MASTER1
		cfg.vm.hostname = "master1"
		cfg.vm.provider "virtualbox" do |vbox|
			vbox.memory = "4048"
		end
		cfg.vm.provision "shell", inline: nodeInit
		cfg.vm.provision "shell", inline: <<-SHELL
			sudo sed -i 's@ExecStart=/usr/bin/kubelet.\\+@\\0 --node-ip=#{IP_MASTER1}@' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
		SHELL
		cfg.vm.provision :reload
		cfg.vm.provision "shell", inline: <<-SHELL
			set -ex
			sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=#{IP_MASTER1} | tee /vagrant/kube-init.log

			# copy admin connectivity settings to common /vagrant folder for later usage
			mkdir -p /vagrant/.kube
			rm -rf /vagrant/.kube/*
			sudo cp -i /etc/kubernetes/admin.conf /vagrant/.kube/admin.conf
			
			# setup user shell
			mkdir -p /home/vagrant/.kube
			sudo cp -i /vagrant/.kube/admin.conf /home/vagrant/.kube/config
			sudo chown vagrant:vagrant /home/vagrant/.kube/config

			sudo -u vagrant bash -c "curl https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml | kubectl apply -f -"
		SHELL
	end

	
    (1..NODE_COUNT).each do |n| 
		vmname = "node%s" % n
		nodeIp = IP_NODE_TEMPL % (n + 20)

		config.vm.define vmname do |cfg|	
			cfg.vm.box = "bento/centos-7.4"
			cfg.vm.network "private_network", ip: nodeIp
			cfg.vm.hostname = vmname
			cfg.vm.provider :virtualbox do |vbox|
				vbox.memory = "4048"
			end

 			cfg.vm.provision "shell", inline: nodeInit
			cfg.vm.provision "shell", inline: <<-SHELL
				sudo sed -i 's@ExecStart=/usr/bin/kubelet.\\+@\\0 --node-ip=#{nodeIp}@' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
			SHELL

			cfg.vm.provision :reload
			cfg.vm.provision "shell", inline: <<-SHELL
				set -ex
				# join node to cluster
				sudo bash -c 'eval $(cat /vagrant/kube-init.log | grep "kubeadm join")'

				# setup user shell
				mkdir -p /home/vagrant/.kube
				sudo cp -i /vagrant/.kube/admin.conf /home/vagrant/.kube/config
				sudo chown vagrant:vagrant /home/vagrant/.kube/config
			SHELL
		end
    end 
end


