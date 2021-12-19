# Provisioning Pod Network Routes

External Loadbalancer를 구성하기 위하여 PureLB를 설치하도록 하겠습니다.

## FRRouting 설치

PureLB를 BGP모드로 동작시키고 있으며, BGP로 전파가 제대로 되는지 확인 하기 위해 FRRouting을 `k8s-router`에 설치해서 BGP를 연결해 봅니다.
또한 빠른 컨버전스를 위해 BFD를 설정합니다.

접속 : `k8s-router`

```
FRRVER="frr-stable"
curl -O https://rpm.frrouting.org/repo/$FRRVER-repo-1-0.el8.noarch.rpm
dnf -y install ./$FRRVER*
dnf -y install frr frr-pythontools
```

Daemon 설정
```
sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sed -i 's/^bfdd=no/bfdd=yes/' /etc/frr/daemons
sed -i 's/^vtysh_enable=no/vtysh_enable=yes/' /etc/frr/daemons
sed -i 's/^#MAX_FDS=1024/MAX_FDS=1024/' /etc/frr/daemons
```

BGP 설정
```
cat << EOF | tee /etc/frr/frr.conf
frr version 8.0
frr defaults traditional
hostname localhost.localdomain
log syslog informational
!
router bgp 64501
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor k8s-ipv4 peer-group
 neighbor k8s-ipv4 remote-as 64500
 neighbor k8s-ipv6 peer-group
 neighbor k8s-ipv6 remote-as 64500
 bgp listen range 192.168.56.0/22 peer-group k8s-ipv4
 bgp listen range 192:168:56::/64 peer-group k8s-ipv6
 !
 address-family ipv4 unicast
  neighbor k8s-ipv4 activate
  neighbor k8s-ipv4 soft-reconfiguration inbound
 exit-address-family
 address-family ipv6 unicast
  neighbor k8s-ipv6 activate
  neighbor k8s-ipv6 soft-reconfiguration inbound
 exit-address-family
exit
!
line vty
!
EOF
```

FRR 재시작
```
systemctl restart frr
```




## Download PureLB

접속 : `k8s-controller` 

```
curl https://gitlab.com/api/v4/projects/purelb%2Fpurelb/packages/generic/manifest/0.0.1/purelb-complete.yaml -o purelb.yaml

```

### Install PurlLB

Kubernetes의 최종 일관성 아키텍처로 인해 이 매니페스트의 첫번째에서는 LBNodeAgent가 실패하게 됩니다. 이는 매니페스트가 사용자 지정 리소스 정의를 정의하고 해당 정의를 사용해서 리소스를 생성하기 때문에 발생합니다. 그리하여 매니페스트를 다시 적용하면 kubernetes가 사용자 지정 리소스 정의를 처리하기 때문에 정상적으로 설치가 완료 됩니다.

```
kubectl apply -f purelb.yaml
```

> 최초 apply
```
namespace/purelb created
customresourcedefinition.apiextensions.k8s.io/lbnodeagents.purelb.io created
customresourcedefinition.apiextensions.k8s.io/servicegroups.purelb.io created
serviceaccount/allocator created
serviceaccount/lbnodeagent created
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/allocator created
podsecuritypolicy.policy/lbnodeagent created
role.rbac.authorization.k8s.io/pod-lister created
clusterrole.rbac.authorization.k8s.io/purelb:allocator created
clusterrole.rbac.authorization.k8s.io/purelb:lbnodeagent created
rolebinding.rbac.authorization.k8s.io/pod-lister created
clusterrolebinding.rbac.authorization.k8s.io/purelb:allocator created
clusterrolebinding.rbac.authorization.k8s.io/purelb:lbnodeagent created
deployment.apps/allocator created
daemonset.apps/lbnodeagent created
error: unable to recognize "purelb.yaml": no matches for kind "LBNodeAgent" in version "purelb.io/v1"
```

> 두번째 apply
```
namespace/purelb unchanged
customresourcedefinition.apiextensions.k8s.io/lbnodeagents.purelb.io configured
customresourcedefinition.apiextensions.k8s.io/servicegroups.purelb.io configured
serviceaccount/allocator unchanged
serviceaccount/lbnodeagent unchanged
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/allocator configured
podsecuritypolicy.policy/lbnodeagent configured
role.rbac.authorization.k8s.io/pod-lister unchanged
clusterrole.rbac.authorization.k8s.io/purelb:allocator unchanged
clusterrole.rbac.authorization.k8s.io/purelb:lbnodeagent unchanged
rolebinding.rbac.authorization.k8s.io/pod-lister unchanged
clusterrolebinding.rbac.authorization.k8s.io/purelb:allocator unchanged
clusterrolebinding.rbac.authorization.k8s.io/purelb:lbnodeagent unchanged
deployment.apps/allocator unchanged
daemonset.apps/lbnodeagent unchanged
lbnodeagent.purelb.io/default created
```

## Verification PureLB

```
kubectl get pods -n purelb 
```

```
NAME                         READY   STATUS    RESTARTS   AGE
allocator-86c57c6d6c-rtrv5   1/1     Running   0          2m
lbnodeagent-4js2x            1/1     Running   0          2m
lbnodeagent-gxhph            1/1     Running   0          2m
lbnodeagent-ms6jp            1/1     Running   0          2m
```

## Initial Configuration

PureLB에서 할당 할 IPv4/IPv6 IP를 설정합니다.

```
cat << EOF | kubectl apply -f -
apiVersion: purelb.io/v1
kind: ServiceGroup
metadata:
  name: ipv6-routed
  namespace: purelb
spec:
  local:
    aggregation: /128
    pool: 2001:470:8bf5:2:2::1-2001:470:8bf5:2:2::ffff
    subnet: 2001:470:8bf5:2::/64

---
apiVersion: purelb.io/v1
kind: ServiceGroup
metadata:
  name: ipv4-routed
  namespace: purelb
spec:
  local:
    aggregation: /32
    pool: 172.30.200.155-172.30.200.160
    subnet: 172.30.200.0/24
EOF
```

## Verification PureLB IPAM

```
kubectl get servicegroup -n purelb
```

>output

```
NAME          AGE
ipv4-routed   21s
ipv6-routed   21s
```

## PureLB BGP 설정

PureLB는 BGP 연동을 위해서 별도로 bird를 daemonset으로 배포하게 되어 있습니다.
bird를 배포하기 위한 별도의 namespace, configmap, daemonset을 생성하도록 하겠습니다.

router namespace 생성

```
kubectl create namespace router
```

configmap 생성

configmap 생성은 cat으로 진행시 문제가 있어서 vi로 직접 파일을 만듭니다.

```
vi bird.configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: bird-cm
  namespace: router
data:
  bird.conf: |-
    # Basic configuration for PureLB Virtual Interface

    # Configure logging
    log stderr all;
    #log "/var/log/bird.log" { debug, trace, info, remote, warning, error, auth, fatal, bug };

    # birdvars is created by docker_entrypoint.  It reads env variables for use in configuration
    include "/usr/local/include/birdvars.conf";

    # Router ID set using birdvars
    router id k8sipaddr;

    watchdog warning 5 s;
    watchdog timeout 30 s;

    # The Device protocol is required, it provide a mechanism for getting 
    # information from the Linux kernel
    protocol device { 
      scan time 10; 
    }

    # The direct protocol is used to import the routes added by PureLb to kube-lb0
    protocol direct {
      ipv4;
      ipv6;
      interface "kube-lb0";
    }

    # The kernel protocol is used to learn and import kernel routes. 
    # These are needed to establish connectivity on the local network
    # Note that no routes from Birds are exported to the kernel, export defaults to false.
    protocol kernel {
      ipv4;
      scan time 10;		
      learn;
    }

    protocol kernel {
      ipv6;
      scan time 10;
      learn;
    }

    # Filter to drop all routes recieved from BGP
    filter bgp_reject {
      reject;
    }

    # BGP example, explicit name 'uplink1' is used instead of default 'bgp1'
    protocol bgp uplink1 {
      description "Example Peer";
      local k8sipaddr as 64500;
      neighbor 192.168.56.100 as 64501 external;
      direct;
      ipv4 {                    # IPv4 unicast (1/1)
        # RTS_DEVICE matches routes added to kube-lb0 by protocol device
        export where source ~ [ RTS_STATIC, RTS_BGP, RTS_DEVICE ];
        import filter bgp_reject; # we are only advertizing
      };
    }
    protocol bgp uplink0 {
      description "Example Peer";
      local as 64500;
      neighbor 192:168:56::100 as 64501 external;
      direct;
      ipv6 {                    # IPv4 unicast (1/1)
        # RTS_DEVICE matches routes added to kube-lb0 by protocol device
        export where source ~ [ RTS_STATIC, RTS_BGP, RTS_DEVICE ];
        import filter bgp_reject; # we are only advertizing
      };

    }



kubectl apply -f bird.configmap
```

bird daemonset 생성

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: bird
  namespace: router
spec:
  selector:
    matchLabels:
      app: bird
  template:
    metadata:
      labels:
        app: bird
    spec:
      containers:
      - name: bird
        env:
        - name: BIRD_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: BIRD_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: BIRD_ML_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.gitlab.com/purelb/bird_router:latest
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 500m
            memory: 250Mi
          requests:
            cpu: 500m
            memory: 250Mi
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: false
        volumeMounts:
        - name: bird-cm
          mountPath: /usr/local/etc
      volumes:
      - name: bird-cm
        configMap:
          name: bird-cm
      hostNetwork: true
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
EOF
```



## Verification

BGP 및 BFD 연결 상태에 대해서 확인 해 봅니다.

접속 : `k8s-router`

frr 접속
```
vtysh
```

ipv4 bgp 상태 확인
```
show ip bgp summary
```

>output

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 192.168.57.100, local AS number 64501 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 2, using 1446 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
*192.168.56.21  4      64500        23        19        0    0    0 00:16:58            0        0 N/A
*192.168.56.22  4      64500        23        19        0    0    0 00:16:58            0        0 N/A

Total number of neighbors 2
* - dynamic neighbor
2 dynamic neighbor(s), limit 100
```

ipv6 bgp 상태 확인
```
show ip bgp ipv6 summary
```

>output

```
IPv6 Unicast Summary (VRF default):
BGP router identifier 192.168.57.100, local AS number 64501 vrf-id 0
BGP table version 4
RIB entries 1, using 184 bytes of memory
Peers 2, using 1446 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
*192:168:56::21 4      64500        34        24        0    0    0 00:17:39            1        1 N/A
*192:168:56::22 4      64500        31        24        0    0    0 00:17:40            0        1 N/A

Total number of neighbors 2
* - dynamic neighbor
2 dynamic neighbor(s), limit 100
```




### BFD 설정 (Option)

초기에 BFD를 동시에 설정 시에 session ID가 다르다고 오류가 지속적으로 발생하며 BGP가 연결되지 않습니다.

그러나 BGP가 연결 된 후 BFD를 설정하면 문제없이 연동이 되고 있습니다. 이 부분은 아직 제대로 확인 되지 않았으며, 별도로 설정하는 방법으로 테스트를 진행합니다.

접속 : `k8s-contoller`

`bird.configmap`파일에 다음 내용들을 추가 합니다.

```
    protocol bfd {
          interface "enp0s*" {
              interval 100ms;
              multiplier 5;
          };
    }

    protocol bgp uplink1 {
      bfd on;
    }
    protocol bgp uplink0 {
      bfd on;
    }
```

설정 적용
```
kubectl apply -f bird.configmap
```

---

접속 : `k8s-router`

frr에 bfd 설정을 합니다.
```
vtysh
config terminal

router bgp 64501
 neighbor k8s-ipv4 bfd
 neighbor k8s-ipv6 bfd
 exit
!
bfd
 profile peer
  detect-multiplier 5
  transmit-interval 100
  receive-interval 100
 exit
 !
 peer 192.168.56.21
  profile peer
 exit
 !
 peer 192.168.56.22
  profile peer
 exit
 !
 peer 192:168:56::21
  profile peer
 exit
 !
 peer 192:168:56::22
  profile peer
 exit

write memory
```

### BFD Verification

bfd 상태 확인
```
show bfd peer 192.168.56.21
```

>output

```
BFD Peer:
	peer 192.168.56.21 vrf default
		ID: 1075829884
		Remote ID: 20040028
		Active mode
		Status: up
		Uptime: 12 minute(s), 44 second(s)
		Diagnostics: ok
		Remote diagnostics: ok
		Peer Type: configured
		Local timers:
			Detect-multiplier: 5
			Receive interval: 100ms
			Transmission interval: 100ms
			Echo receive interval: 50ms
			Echo transmission interval: disabled
		Remote timers:
			Detect-multiplier: 5
			Receive interval: 100ms
			Transmission interval: 100ms
			Echo receive interval: disabled
```

Next: [Validation Dualstack](08-validation-dualstack.md)