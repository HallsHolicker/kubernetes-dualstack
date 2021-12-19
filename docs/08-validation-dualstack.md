# Validation Dualstack

구성된 환경으로 dualstack이 제대로 동작하는지 확인 해 보겠습니다.

## Deploy Pods

### webpod deploy
IPv6가 가능 한 webpod Deployment를 배포하겠습니다.

접속 : `k8s-controller`

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod-deployment
spec:
  selector:
    matchLabels:
      app: webpod
  replicas: 2
  template:
    metadata:
      labels:
        app: webpod
    spec:
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
EOF
```

### Verification Deployment

deployment 확인

```
kubectl get deploy
```

>output

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
webpod-deployment   2/2     2            2           23s
```

pods 확인

```
kubectl get pods -o wide
```

>output

```
NAME                                 READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
webpod-deployment-7b65f8c6cf-9drl5   1/1     Running   0          34s   10.244.194.68   k8s-worker1   <none>           <none>
webpod-deployment-7b65f8c6cf-z5nn7   1/1     Running   0          34s   10.244.126.6    k8s-worker2   <none>           <none>
```

pod 상세 정보 확인

```
kubectl describe pods webpod-deployment-7b65f8c6cf-9drl5
```

>output 

```
Name:         webpod-deployment-7b65f8c6cf-9drl5
Namespace:    default
Priority:     0
Node:         k8s-worker1/192.168.56.21
Start Time:   Mon, 20 Dec 2021 01:36:51 +0900
Labels:       app=webpod
              pod-template-hash=7b65f8c6cf
Annotations:  cni.projectcalico.org/containerID: 4e1f693d9b276d398283c03c4778e376b5447bb074b042a37edc38259ac0d998
              cni.projectcalico.org/podIP: 10.244.194.68/32
              cni.projectcalico.org/podIPs: 10.244.194.68/32,2001:db8:42:93:c54a:4054:9104:c243/128
Status:       Running
IP:           10.244.194.68
IPs:
  IP:           10.244.194.68
  IP:           2001:db8:42:93:c54a:4054:9104:c243
Controlled By:  ReplicaSet/webpod-deployment-7b65f8c6cf
Containers:
  webpod:
    Container ID:   containerd://0fd646dc781e3f1d4af272eeda0567fa6e65b5498f1a8549491176f9f90f1d63
    Image:          traefik/whoami
    Image ID:       docker.io/traefik/whoami@sha256:615ee4ae89e143eb4d8261a2699ad4659b5bcc70156928e1a721e088f458a38d
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 20 Dec 2021 01:36:54 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kn96c (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-kn96c:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  50s   default-scheduler  Successfully assigned default/webpod-deployment-7b65f8c6cf-9drl5 to k8s-worker1
  Normal  Pulling    49s   kubelet            Pulling image "traefik/whoami"
  Normal  Pulled     47s   kubelet            Successfully pulled image "traefik/whoami" in 1.973552345s
  Normal  Created    47s   kubelet            Created container webpod
  Normal  Started    47s   kubelet            Started container webpod
```

기본적인 pod의 정보에는 IPv6의 주소가 없는 것 처럼 보이지만, 상세 정보를 들어가 보면 IPv6 주소가 할당되어 있는것이 확인 가능합니다.


## Deploy Service

Servie를 Loadbalancer로 배포하여 PurlLB에서 제대로 IPv4, IPv6 주소를 할당 받는지 확인해 보겠습니다.
현재, PureLB는 Service에 IPv4/IPv6를 동시에 할당을 하지 않습니다. 그리하여, IPv4/IPv6를 쓰기 위해서는 각각 Service를 만들어야 합니다.

접속 : `k8s-controller`

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  annotations:
    purelb.io/allocated-by: PureLB
    purelb.io/allocated-from: ipv4-routed
    purelb.io/service-group: ipv4-routed
  creationTimestamp: "2021-09-07T12:29:29Z"
  name: webpod-service
  namespace: default
  resourceVersion: "1471429"
  selfLink: /api/v1/namespaces/default/services/webpod-service
  uid: ca7c289e-bacb-45e0-9b3f-51b67eec7429
spec:
  allocateLoadBalancerNodePorts: true
  externalTrafficPolicy: Local
  healthCheckNodePort: 32083
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 31292
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webpod
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 172.30.200.155

---

apiVersion: v1
kind: Service
metadata:
  annotations:
    purelb.io/allocated-by: PureLB
    purelb.io/allocated-from: ipv6-routed
    purelb.io/service-group: ipv6-routed
  creationTimestamp: "2021-09-07T14:50:55Z"
  name: webpod-service-ip6
  namespace: default
  resourceVersion: "1471462"
  selfLink: /api/v1/namespaces/default/services/webpod-service-ip6
  uid: 9d25ca30-3379-423a-92cb-7c66318be03e
spec:
  allocateLoadBalancerNodePorts: true
  externalTrafficPolicy: Local
  healthCheckNodePort: 30388
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv6
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30091
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webpod
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 2001:470:8bf5:2:2::1
EOF
```


### Verification Service

Service 확인

```
kubectl get svc -o wide
```

>output

```
NAME                 TYPE           CLUSTER-IP            EXTERNAL-IP            PORT(S)        AGE   SELECTOR
kubernetes           ClusterIP      10.96.0.1             <none>                 443/TCP        68m   <none>
webpod-service       LoadBalancer   10.96.67.187          172.30.200.155         80:31292/TCP   6s    app=webpod
webpod-service-ip6   LoadBalancer   2001:db8:42:1::193d   2001:470:8bf5:2:2::1   80:30091/TCP   6s    app=webpod
```


Endpoints 확인

```
kubectl get endpoints
```

>output

```
NAME                 ENDPOINTS                                                                         AGE
kubernetes           192.168.56.11:6443                                                                69m
webpod-service       10.244.126.6:80,10.244.194.68:80                                                  20s
webpod-service-ip6   [2001:db8:42:47:5b8f:562b:2aae:7e05]:80,[2001:db8:42:93:c54a:4054:9104:c243]:80   20s
```

BGP 전파 확인

`k8s-router`



```
vtysh
show ip route
show ipv6 route
```

>show ip route output

```
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/100] via 10.0.2.2, enp0s3, 00:41:51
C>* 10.0.2.0/24 is directly connected, enp0s3, 00:41:51
B>* 172.30.200.155/32 [20/0] via 192.168.56.21, enp0s8, weight 1, 00:03:09
  *                          via 192.168.56.22, enp0s8, weight 1, 00:03:09
C>* 192.168.56.0/24 is directly connected, enp0s8, 00:41:51
C>* 192.168.57.0/24 is directly connected, enp0s9, 00:41:51
```

>show ipv6 route output

```
Codes: K - kernel route, C - connected, S - static, R - RIPng,
       O - OSPFv3, I - IS-IS, B - BGP, N - NHRP, T - Table,
       A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 192:168:56::/64 is directly connected, enp0s8, 00:40:48
C>* 192:168:57::/64 is directly connected, enp0s9, 00:40:48
B>* 2001:470:8bf5:2:2::1/128 [20/0] via fe80::a00:27ff:fe53:7114, enp0s8, weight 1, 00:02:06
  *                                 via fe80::a00:27ff:fee5:d93, enp0s8, weight 1, 00:02:06
C * fe80::/64 is directly connected, enp0s9, 00:40:48
C * fe80::/64 is directly connected, enp0s8, 00:40:48
C>* fe80::/64 is directly connected, enp0s3, 00:40:48
```

확인된 정보와 같이 IPv4/IPv6 주소가 전파되고 있는 것을 확인 해 볼 수 있습니다.

### 통신 확인

접속 : `k8s-client`

IPv4 VIP로 통신 테스트

```
curl -s 172.30.200.155
```

>output

```
Hostname: webpod-deployment-7b65f8c6cf-9drl5
IP: 127.0.0.1
IP: ::1
IP: 10.244.194.68
IP: 2001:db8:42:93:c54a:4054:9104:c243
IP: fe80::f040:71ff:feb6:6557
RemoteAddr: 192.168.57.10:60600
GET / HTTP/1.1
Host: 172.30.200.155
User-Agent: curl/7.61.1
Accept: */*
```

IPv6 VIP로 통신 테스트

```
curl -g [2001:470:8bf5:2:2::1]
```

>output

```
Hostname: webpod-deployment-7b65f8c6cf-z5nn7
IP: 127.0.0.1
IP: ::1
IP: 10.244.126.6
IP: 2001:db8:42:47:5b8f:562b:2aae:7e05
IP: fe80::64f6:5aff:fe2e:acae
RemoteAddr: [192:168:57::10]:55578
GET / HTTP/1.1
Host: [2001:470:8bf5:2:2::1]
User-Agent: curl/7.61.1
Accept: */*
```