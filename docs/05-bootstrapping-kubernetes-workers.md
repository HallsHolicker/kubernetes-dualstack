# Bootstrapping the Kubernetes Worker Nodes

이번 구성은 Kubernete worker node들을 부트스트랩 하겠습니다. 각 노드에 다음 구성요소를 설치하겠습니다.
CNI의 경우는 calico를 설치할 것이며, 설치는 [Provisioning Pod Network CNI Calico](09-provisioning-pod-network-cni-calico.md)에서 진행하도록 하겠습니다.

## Prerequisites

이번 실습 명령어는 각 Worker node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Worker node: `k8s-worker-1`, `k8s-worker-2`

```
ssh k8s-worker-1
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.


## Provisioning a Kubernetes Worker Node

kubernetes Repository 추가:

```
cat << EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

### Install the Kubernetes kubeadm, kubelet, kubectl

Kubernetes 설치:

```
sudo dnf -y install kubeadm-1.23.0 kubelet-1.23.0 kubectl-1.23.0 --disableexcludes=kubernetes
```

### Start kubelet

kubelet 실행

```
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

### Create kubeadm-config.yaml

`k8s-controller`에서 나온 정보를 가지고 kubeadm-config.yaml을 생성합니다.

```
cat << EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 192.168.56.11:6443
    token: "fwpnj5.y36a32hppohzmizp"
    caCertHashes:
    - "sha256:aee1fe2af543458fb02600296995f6c27222196b2bdb2cec1b9aa4f42c3fc07f"
    # change auth info above to match the actual token and CA certificate hash for your cluster
nodeRegistration:
  kubeletExtraArgs:
    node-ip: 192.168.56.21,192:168:56:::21  # Worker node IP에 맞게 수정
EOF
```

### Join kubernetes cluster

생성된 kubeadm-config.yaml을 가지고 k8s cluster에 join 합니다.

```
kubeadm join --config kubeadm-config.yaml
```

## Verification

Kubernetes nodes 등록 상태 확인:

`k8s-controller`에서 확인

```
  kubectl get nodes
```

> output

```
NAME            STATUS     ROLES                  AGE     VERSION
k8s-contoller   NotReady   control-plane,master   4m30s   v1.23.0
k8s-worker1     NotReady   <none>                 24s     v1.23.0
k8s-worker2     NotReady   <none>                 17s     v1.23.0
```

아직 네트워크 CNI를 설정하지 않았기 때문에 NotReady 상태로 표시 됩니다.

Next: [Provisioning Pod Network CNI Calico](06-provisioning-pod-network-cni-calico.md)
