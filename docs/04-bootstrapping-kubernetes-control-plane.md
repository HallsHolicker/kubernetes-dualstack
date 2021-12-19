# Bootstrapping the Kubernetes Control Plane

## Prerequisites

이번 구성 명령어는 Controller node에서 진행해야 합니다.
Controller node: `k8s-controller`

```
ssh k8s-controller
```

## Provision the Kubernetes Control Plane

kubernetes Repository 추가:

```
cat << EOF | tee /etc/yum.repos.d/kubernetes.repo
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
dnf -y install kubeadm-1.23.0 kubelet-1.23.0 kubectl-1.23.0 --disableexcludes=kubernetes
```

### Start kubelet

kubelet 실행

```
systemctl enable kubelet
systemctl start kubelet
```

### Create kubeadm-config.yml

kubeadm init를 실행하면 기본은 single-stack으로 동작하여 IPv4만 동작하게 되어 있습니다.
dual-stack으로 동작하기 위해서는 명령어에 다음과 같이 하거나 kubeadm-config.yaml을 만들어서 사용해야 합니다.
`kubeadm init --pod-network-cidr=10.244.0.0/16,2001:db8:42:0::/56 --service-cidr=10.96.0.0/16,2001:db8:42:1::/112`

이번 구성에서는 kubeadm-config.yaml을 사용해서 만들도록 하겠습니다.
```
cat << EOF > kubeadm-config.yaml
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: 10.244.0.0/16,2001:db8:42:0::/56
  serviceSubnet: 10.96.0.0/16,2001:db8:42:1::/112
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.56.11"
  bindPort: 6443
nodeRegistration:
  kubeletExtraArgs:
    node-ip: 192.168.56.11,192:168:56::11
EOF
```

Result:

```
kubeadm-config.yml
```

### Kubernetes Image Pull

Kubernetes에서 사용할 이미지들을 미리 가져오겠습니다.

```

kubeadm config images pull

```

### Kubernetes Contoller 구성

```
kubeadm init --config kubeadm-config.yaml
```

Output에서 나오는 정보중 token / discovery-token-ca-cert-hash 값은 join시에도 사용하기에 따로 메모해 둡니다.
>output

```
~~~
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.56.11:6443 --token fwpnj5.y36a32hppohzmizp \
	--discovery-token-ca-cert-hash sha256:aee1fe2af543458fb02600296995f6c27222196b2bdb2cec1b9aa4f42c3fc07f \
	--control-plane --certificate-key 94ab545d5869f1d9caf5b7a72211aaf1c893da164342eaaf22ead82ffc1ba6c0

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.11:6443 --token fwpnj5.y36a32hppohzmizp \
	--discovery-token-ca-cert-hash sha256:aee1fe2af543458fb02600296995f6c27222196b2bdb2cec1b9aa4f42c3fc07f

```

### Setting kubectl admin.conf

kubectl을 편안하게 사용하게 위해 `k8s-controller`에서 생성된 admin.conf를 이용하여 kubeconfig를 구성합니다.

```
mkdir -p ~/.kube
cp -i /etc/kubernetes/admin.conf ~/.kube/config
```

## Verification

```
kubectl get nodes
```

> output

```
NAME               STATUS     ROLES                  AGE   VERSION
k8s-controller-0   NotReady   control-plane,master   2m   v1.23.0
```

Next: [Bootstrapping the Kubernetes Worker Nodes](05-bootstrapping-kubernetes-workers.md)
