# Provisioning Pod Network Routes

이번 실습에서는 Kubernetes CNI로 calico를 설치하도록 하겠습니다. 또한. External Loadbalancer를 구성하기 위해여 metalLB도 설치하도록 하겠습니다.

## Installing kubernetes CNI calico

이번 작업은 `k8s-controller`에서 진행합니다. 

```
ssh k8s-contoller
```

### Download Calico manifests

```
curl https://docs.projectcalico.org/v3.21/manifests/calico.yaml -o calico.yaml
```

### Calico Version check

```
 less calico.yaml | grep image
```

> output

```
          image: docker.io/calico/cni:v3.21.2
          image: docker.io/calico/cni:v3.21.2
          image: docker.io/calico/pod2daemon-flexvol:v3.21.2
          image: docker.io/calico/node:v3.21.2
          image: docker.io/calico/kube-controllers:v3.21.2
```


### Enalbe dualstack

dualstack을 실행하기 위해 다음값을 추가 및 수정 합니다.
또한 Loadbalancer로 사용하는 PurlLB가 BGP를 사용하기에 calico를 vxlan 모드로 변경합니다.

추가값
```
          "ipam": {
              "type": "calico-ipam",
              "assign_ipv4": "true",
              "assign_ipv6": "true"
          },

```

수정값
```
  calico_backend: "vxlan"

            - name: CALICO_IPV4POOL_IPIP
              value: "Never"
            # Enable or Disable VXLAN on the default IP pool.
            - name: CALICO_IPV4POOL_VXLAN
              value: "Always"

            - name: FELIX_IPV6SUPPORT
              value: "true"

```

삭제값
```
              - -bird-live

              - -bird-ready

```

### Install Calico

```
kubectl apply -f calico.yaml
```

## Verification Calico

Calico가 정상 동작하기까지는 약 2~3분 정도 소요됩니다.
무선 환경에서 배포할 경우 인터넷 환경에 따라 1시간 이상이 소요될 수도 있습니다. -0-

```
kubectl get pods -n kube-system
```

> output

```
NAME                                       READY   STATUS    RESTARTS      AGE
calico-kube-controllers-647d84984b-z247m   1/1     Running   0          3m32s
calico-node-99v2g                          1/1     Running   0          3m32s
calico-node-tpk7s                          1/1     Running   0          3m32s
calico-node-vkkxz                          1/1     Running   0          3m32s
```


Next: [Provisioning Loadbalancer Network PureLB](07-provisioning-lb-network-purelb.md)
