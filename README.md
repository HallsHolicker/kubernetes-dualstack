# Kubernetes Dualstack Onprem Labs

이 가이드는 온프렘에서 K8S 1.23부터 GA가된 Dualstack을 테스트하기 위한 환경을 구현하는 방법을 다룹니다.

## Architecture

![architecture](docs/images/kubernetes_onprem_lab.png "architecture")

* CNI는 Calico를 사용하였습니다.
* On-premise의 환경에서 사용하는 LoadBalancer는 PureLB로 구성하였습니다.


## Cluster Details

* [kubernetes](https://github.com/kubernetes/kubernetes) v1.23.0
* [containerd](https://github.com/containerd/containerd) v1.4.12
* [calico](https://github.com/projectcalico/calico) v3.21.2
* [PureLB](https://purelb.gitlab.io/docs/) v0.7.0
* [FRRouting](https://frrouting.org/) v7.5.1

## Labs

테스트 환경은 On-premise와 비슷하게 테스트하기 위해 가상머신에서 테스트를 진행하며, 가상머신은 Virtualbox를 사용하였습니다.

* [Prerequisites](docs/01-prerequisites.md)
* [Provisioning Compute Resources](docs/02-compute-resources.md)
* [Bootstrapping Container Runtime](docs/03-bootstrapping-container-runtime.md)
* [Bootstrapping the Kubernetes Control Plane](docs/04-bootstrapping-kubernetes-control-plane.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/05-bootstrapping-kubernetes-workers.md)
* [Provisioning Pod Network CNI Calico](docs/06-provisioning-pod-network-cni-calico.md)
* [Provisioning Loadbalancer Network PureLB](docs/07-provisioning-lb-network-purelb.md)
* [Validation DualStack](docs/08-validation-dualstack.md)


## 참고 사이트
* [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
* [IPv6 nginx](https://github.com/leblancd/kube-v6)
