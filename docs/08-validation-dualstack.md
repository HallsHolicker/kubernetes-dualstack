# Validation Dualstack

구성된 환경으로 dualstack이 제대로 동작하는지 확인 해 보겠습니다.

## Deploy Pods

### nginx deploy
IPv6가 가능 한 Nginx Deployment를 배포하겠습니다.

접속 : `k8s-controller`

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: diverdane/nginxdualstack:1.0.0
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
nginx-deployment   2/2     2            2           23s
```

pods 확인

```
kubectl get pods -o wide
```

>output

```
NAME                                READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
nginx-deployment-76d96cc579-58wmf   1/1     Running   0          53s   10.244.126.4    k8s-worker2   <none>           <none>
nginx-deployment-76d96cc579-z7gd7   1/1     Running   0          53s   10.244.194.66   k8s-worker1   <none>           <none>
```

pod 상세 정보 확인

```
kubectl describe pods nginx-deployment-76d96cc579-58wmf
```

>output 

```
Name:         nginx-deployment-76d96cc579-58wmf
Namespace:    default
Priority:     0
Node:         k8s-worker2/192.168.56.22
Start Time:   Mon, 20 Dec 2021 01:03:04 +0900
Labels:       app=nginx
              pod-template-hash=76d96cc579
Annotations:  cni.projectcalico.org/containerID: be33f441f30a908ad967cc507c8f938e9f8c4bedf889269d43d207a4d5b72f95
              cni.projectcalico.org/podIP: 10.244.126.4/32
              cni.projectcalico.org/podIPs: 10.244.126.4/32,2001:db8:42:47:5b8f:562b:2aae:7e03/128
Status:       Running
IP:           10.244.126.4
IPs:
  IP:           10.244.126.4
  IP:           2001:db8:42:47:5b8f:562b:2aae:7e03
Controlled By:  ReplicaSet/nginx-deployment-76d96cc579
Containers:
  nginx:
    Container ID:   containerd://dcd039c3e436f6041a48e8af499e9bef1a18f931ba9be93e9cc12052a91c140d
    Image:          diverdane/nginxdualstack:1.0.0
    Image ID:       docker.io/diverdane/nginxdualstack@sha256:681393039246e4328778ff566492606220ee8bdc1829fc8488fc1574b9e0f543
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 20 Dec 2021 01:03:20 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hlrv8 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-hlrv8:
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
  Normal  Scheduled  109s  default-scheduler  Successfully assigned default/nginx-deployment-76d96cc579-58wmf to k8s-worker2
  Normal  Pulling    107s  kubelet            Pulling image "diverdane/nginxdualstack:1.0.0"
  Normal  Pulled     92s   kubelet            Successfully pulled image "diverdane/nginxdualstack:1.0.0" in 15.331683071s
  Normal  Created    92s   kubelet            Created container nginx
  Normal  Started    92s   kubelet            Started container nginx
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
  name: nginx-service
  namespace: default
  resourceVersion: "1471429"
  selfLink: /api/v1/namespaces/default/services/nginx-service
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
    app: nginx
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
  name: nginx-service-ip6
  namespace: default
  resourceVersion: "1471462"
  selfLink: /api/v1/namespaces/default/services/nginx-service-ip6
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
    app: nginx
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
NAME                TYPE           CLUSTER-IP            EXTERNAL-IP            PORT(S)        AGE   SELECTOR
kubernetes          ClusterIP      10.96.0.1             <none>                 443/TCP        43m   <none>
nginx-service       LoadBalancer   10.96.242.195         172.30.200.155         80:31292/TCP   4s    app=nginx
nginx-service-ip6   LoadBalancer   2001:db8:42:1::d61d   2001:470:8bf5:2:2::1   80:30091/TCP   4s    app=nginx
```


Endpoints 확인

```
kubectl get endpoints
```

>output

```
NAME                TYPE           CLUSTER-IP            EXTERNAL-IP            PORT(S)        AGE   SELECTOR
kubernetes          ClusterIP      10.96.0.1             <none>                 443/TCP        43m   <none>
nginx-service       LoadBalancer   10.96.242.195         172.30.200.155         80:31292/TCP   4s    app=nginx
nginx-service-ip6   LoadBalancer   2001:db8:42:1::d61d   2001:470:8bf5:2:2::1   80:30091/TCP   4s    app=nginx
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
<!DOCTYPE html>
<html>
<head>
<title>Kubernetes IPv6 nginx</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx on <span style="color:  #C70039">IPv6</span> Kubernetes!</h1>
<p>Pod: nginx-deployment-76d96cc579-z7gd7</p>
</body>
</html>
```

IPv6 VIP로 통신 테스트

```
curl -g [2001:470:8bf5:2:2::1]
```

>output

```
<!DOCTYPE html>
<html>
<head>
<title>Kubernetes IPv6 nginx</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx on <span style="color:  #C70039">IPv6</span> Kubernetes!</h1>
<p>Pod: nginx-deployment-76d96cc579-58wmf</p>
</body>
</html>
```