# Provisioning Compute Resources

Kubernetes는 control plane과 Container가 실행되는 Worker node를 호스팅하기 위한 장비가 필요합니다.
안전하고 가용성이 높은 Kubernetes Cluster를 실행하는데 필요한 컴퓨팅 리소스를 프로비저닝합니다.

## Compute Instances

이미지는 `centos8-stream`으로 별도로 만든 이미지를 사용하였습니다.

Control Plane Node 3개, Worker Node 3개, Router 1개, Client 1개를 생성하겠습니다.

```
cat << EOF > Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "hallsholicker/centos8-stream-k8s"

  config.vm.define "k8s-controller" do |controller|
    controller.vm.network "private_network", ip: "192.168.56.11"
    controller.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-controller"
      v.memory = 2048
      v.cpus = 2
      v.linked_clone = true
    end
    controller.vm.provision "shell", inline: <<-SHELL
      echo "sudo -i" >> .bashrc
      sudo -i
      sudo hostnamectl set-hostname k8s-contoller
      sudo echo "192.168.56.11 k8s-contoller" >> /etc/hosts
      sudo echo "192.168.56.21 k8s-worker1" >> /etc/hosts
      sudo echo "192.168.56.22 k8s-worker2" >> /etc/hosts
      sudo echo "IPV6INIT=yes" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6ADDR=192:168:56::11/64" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6_DEFAULTGW=192:168:56::100" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "default via 192:168:56::100" >> /etc/sysconfig/network-scripts/route6-enp0s8
      sudo ip address add 192:168:56::11/64 dev enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip -6 route add ::/0 via 192:168:56::100
    SHELL
  end

  config.vm.define "k8s-worker-1" do |worker1|
    worker1.vm.network "private_network", ip: "192.168.56.21"
    worker1.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-worker-1"
      v.memory = 1536
      v.cpus = 2
      v.linked_clone = true
    end
    worker1.vm.provision "shell", inline: <<-SHELL
      echo "sudo -i" >> .bashrc
      sudo -i
      sudo hostnamectl set-hostname k8s-worker1
      sudo echo "192.168.56.11 k8s-contoller" >> /etc/hosts
      sudo echo "192.168.56.21 k8s-worker1" >> /etc/hosts
      sudo echo "192.168.56.22 k8s-worker2" >> /etc/hosts
      sudo echo "IPV6INIT=yes" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6ADDR=192:168:56::21/64" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6_DEFAULTGW=192:168:56::100" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "default via 192:168:56::100" >> /etc/sysconfig/network-scripts/route6-enp0s8
      sudo ip address add 192:168:56::21/64 dev enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip -6 route add ::/0 via 192:168:56::100

    SHELL
  end

  config.vm.define "k8s-worker-2" do |worker2|
    worker2.vm.network "private_network", ip: "192.168.56.22"
    worker2.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-worker-2"
      v.memory = 1536
      v.cpus = 2
      v.linked_clone = true
    end
    worker2.vm.provision "shell", inline: <<-SHELL
      echo "sudo -i" >> .bashrc
      sudo -i
      sudo hostnamectl set-hostname k8s-worker2
      sudo echo "192.168.56.11 k8s-contoller" >> /etc/hosts
      sudo echo "192.168.56.21 k8s-worker1" >> /etc/hosts
      sudo echo "192.168.56.22 k8s-worker2" >> /etc/hosts
      sudo echo "IPV6INIT=yes" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6ADDR=192:168:56::22/64" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6_DEFAULTGW=192:168:56::100" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "default via 192:168:56::100" >> /etc/sysconfig/network-scripts/route6-enp0s8
      sudo ip address add 192:168:56::22/64 dev enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip -6 route add ::/0 via 192:168:56::100
    SHELL
  end

  config.vm.define "k8s-router" do |router|
    router.vm.network "private_network", ip: "192.168.56.100"
    router.vm.network "private_network", ip: "192.168.57.100"
    router.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-router"
      v.memory = 1024
      v.cpus = 1
      v.linked_clone = true
    end
    router.vm.provision "shell", inline: <<-SHELL
      echo "sudo -i" >> .bashrc
      sudo -i
      sudo echo "net.ipv4.ip_forward=1" > /etc/sysctl.conf
      sudo echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf
      sudo ip address add 192:168:56::100/64 dev enp0s8
      sudo ip address add 192:168:57::100/64 dev enp0s9
      sudo sysctl -p

      SHELL
  end

  config.vm.define "k8s-client" do |client|
    client.vm.network "private_network", ip: "192.168.57.10"
    client.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-client"
      v.memory = 1024
      v.cpus = 1
      v.linked_clone = true
    end
    client.vm.provision "shell", inline: <<-SHELL
      echo "sudo -i" >> .bashrc
      sudo -i
      sudo echo "IPV6INIT=yes" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6ADDR=192:168:57::10/64" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "IPV6_DEFAULTGW=192:168:57::100" >> /etc/sysconfig/network-scripts/ifcfg-enp0s8
      sudo echo "192.168.56.0/24 via 192.168.57.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "172.30.200.0/24 via 192.168.57.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "default via 192:168:57::100" >> /etc/sysconfig/network-scripts/route6-enp0s8
      sudo ip address add 192:168:57::10/64 dev enp0s8
      sudo ip route add 192.168.56.0/24 via 192.168.57.100
      sudo ip route add 172.30.200.0/24 via 192.168.57.100
      sudo ip -6 route add ::/0 via 192:168:57::100
    SHELL
  end
end

EOF


vagrant up
```

Next: [Bootstrapping Container Runtime](03-bootstrapping-container-runtime)
