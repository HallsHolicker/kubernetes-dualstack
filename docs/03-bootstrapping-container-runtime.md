# Bootstrapping Container Runtime

이번 구성은 Kubernetes에서 사용할 Container Runtime인 Containerd를 구성합니다.

## Prerequisites

이번 실습 명령어는 각 Controller / Worker node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Controller node: `k8s-controller`
Worker node: `k8s-worker-1`, `k8s-worker-2`

```
ssh k8s-controller
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.


### Load Kernel Module

Containerd를 설치하기 전에 필요한 kernel module을 올립니다.

```
modprobe br_netfilter
modprobe overlay
```

모듈 동작 확인
```
lsmod | egrep "br_netfilter|overlay"
```

Result:

```
overlay               135168  0
br_netfilter           24576  0
bridge                192512  1 br_netfilter
```

### Kernel 설정 변경

> kubernetes

```
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

> Calico

```
cat <<EOF | tee /etc/sysctl.d/calico.conf
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.forwarding = 1
EOF

sysctl --system
```

> PureLB

```
cat <<EOF | tee /etc/sysctl.d/purelb.conf
net.ipv4.conf.all.arp_filter=1
EOF

sysctl --system
```

### Install Required packages

Containerd를 설치하는데 필요한 패키지를 설치하겠습니다.

```
dnf -y install device-mapper-persistent-data lvm2 iproute-tc

```

### Add Docker-ce Repositery

Containerd를 설치하기 위한 Repositery 등록

```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```


### Install Containerd

Containerd 설치

```

dnf -y install containerd.io

```


### Configure containerd

```

containerd config default > /etc/containerd/config.toml

```

### Configure cgroup driver systemd

위에 생성한 config.toml 파일에 다음 값을 입력합니다.
`vi /etc/containerd/config.toml`

```
sed -i'' -r -e "/containerd.runtimes.runc.options/a\            SystemdCgroup = true" /etc/containerd/config.toml

grep -B 1 "SystemdCgroup" /etc/containerd/config.toml
```

>output 

```
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```


### Start containerd

```

systemctl restart containerd

```

### Status Containerd

```
systemctl status containerd
```

> output

```
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2021-11-29 17:09:11 KST; 3min 26s ago
     Docs: https://containerd.io
  Process: 7564 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
 Main PID: 7566 (containerd)
    Tasks: 9
   Memory: 21.5M
   CGroup: /system.slice/containerd.service
           └─7566 /usr/bin/containerd
```

Next: [Bootstrapping the Kubernetes Control Plane](04-bootstrapping-kubernetes-control-plane.md)
