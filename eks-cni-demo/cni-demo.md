# CNI DEMO

##  Pre-Assumptions

* You have created a EKS cluster (e.g. using `eksctl`)

* You can ssh into your worker nodes

## Pod Netowrk Stack Internal

```
kubectl get node
NAME                              STATUS    ROLES     AGE       VERSION
ip-192-168-112-98.ec2.internal    Ready     <none>    14m       v1.10.3
ip-192-168-167-201.ec2.internal   Ready     <none>    14m       v1.10.3
ip-192-168-242-227.ec2.internal   Ready     <none>    14m       v1.10.3
```

### Create Pods

```
kubectl apply -f worker_hello.yaml
```

### Show Pods

```
kubectl get pod -o wide
NAME                            READY     STATUS    RESTARTS   AGE       IP                NODE
worker-hello-5d9b798f74-72q5t   1/1       Running   0          27s       192.168.130.206   ip-192-168-167-201.ec2.internal
worker-hello-5d9b798f74-cbgz5   1/1       Running   0          27s       192.168.247.50    ip-192-168-242-227.ec2.internal
worker-hello-5d9b798f74-f9pkn   1/1       Running   0          27s       192.168.157.205   ip-192-168-167-201.ec2.internal
worker-hello-5d9b798f74-fv8jw   1/1       Running   0          27s       192.168.207.203   ip-192-168-242-227.ec2.internal
worker-hello-5d9b798f74-g86wg   1/1       Running   0          27s       192.168.119.223   ip-192-168-112-98.ec2.internal
worker-hello-5d9b798f74-hspdj   1/1       Running   0          27s       192.168.121.243   ip-192-168-112-98.ec2.internal
worker-hello-5d9b798f74-j8vtf   1/1       Running   0          27s       192.168.82.156    ip-192-168-112-98.ec2.internal
worker-hello-5d9b798f74-mjxnn   1/1       Running   0          27s       192.168.238.188   ip-192-168-242-227.ec2.internal
worker-hello-5d9b798f74-sfgtb   1/1       Running   0          27s       192.168.141.2     ip-192-168-167-201.ec2.internal
worker-hello-5d9b798f74-wcsgs   1/1       Running   0          27s       192.168.224.202   ip-192-168-242-227.ec2.internal
```
### Inside Pod

```
 kubectl exec -ti worker-hello-5d9b798f74-72q5t sh
```

#### showing Pod IP

```
ifconfig
eth0      Link encap:Ethernet  HWaddr 2E:9F:31:AB:BB:33  
          inet addr:192.168.130.206  Bcast:192.168.130.206  Mask:255.255.255.255
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:936 (936.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

#### showing pod route table

```
ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0 scope link 
```

#### showing pod arp table

```
arp -a
? (169.254.1.1) at 26:50:08:03:3e:a6 [ether] PERM on eth0
? (169.254.1.1) at 26:50:08:03:3e:a6 [ether] PERM on eth0
```

### ssh into worker

***make sure opening ssh port !***

#### showing route table to Pod traffic
```
ip route
default via 192.168.128.1 dev eth0 
169.254.169.254 dev eth0 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
192.168.128.0/18 dev eth0 proto kernel scope link src 192.168.167.201 
192.168.130.206 dev eni61b9c53e3d2 scope link <------ Pod
192.168.141.2 dev eni3fd21969a5d scope link 
192.168.157.205 dev enid01321d94b0 scope link 
```

#### showing route table from Pod Traffic

```
# mac address matches Pod's ARP entry, 

ifconfig eni61b9c53e3d2
eni61b9c53e3d2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::2450:8ff:fe03:3ea6  prefixlen 64  scopeid 0x20<link>
        ether 26:50:08:03:3e:a6  txqueuelen 0  (Ethernet) <-- match Pod's ARP
        RX packets 2  bytes 126 (126.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15  bytes 1191 (1.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
``` 

```
# choose route table
ip rule show
0:	from all lookup local 
512:	from all to 192.168.141.2 lookup main 
512:	from all to 192.168.157.205 lookup main 
512:	from all to 192.168.130.206 lookup main 
1024:	not from all to 192.168.0.0/16 lookup main 
1536:	from 192.168.157.205 lookup 3 
1536:	from 192.168.130.206 lookup 3 <-- Pod is using route table 3
32766:	from all lookup main 
32767:	from all lookup default  
```

```
# show route table 3

ip route show table 3
default via 192.168.128.1 dev eth2 <-- outgoing eth2
192.168.128.1 dev eth2 scope link
```

#### exam Pod's traffic

```
yum install -y tcpdump
```

```
tcpdump -i eni61b9c53e3d2
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eni61b9c53e3d2, link-type EN10MB (Ethernet), capture size 262144 bytes
01:39:03.397833 IP ip-192-168-130-206.ec2.internal > ip-192-168-247-50.ec2.internal: ICMP echo request, id 3840, seq 0, length 64
01:39:03.398613 IP ip-192-168-247-50.ec2.internal > ip-192-168-130-206.ec2.internal: ICMP echo reply, id 3840, seq 0, length 64
01:39:04.398000 IP ip-192-168-130-206.ec2.internal > ip-192-168-247-50.ec2.internal: ICMP echo request, id 3840, seq 1, length 64
01:39:04.398789 IP ip-192-168-247-50.ec2.internal > ip-192-168-130-206.ec2.internal: ICMP echo reply, id 3840, seq 1, length 64
01:39:08.413836 ARP, Request who-has ip-192-168-130-206.ec2.internal tell ip-192-168-167-201.ec2.internal, length 28
01:39:08.413889 ARP, Reply ip-192-168-130-206.ec2.internal is-at 2e:9f:31:ab:bb:33 (oui Unknown), length 28
```

```
# inside pod
ping 192.168.247.50
PING 192.168.247.50 (192.168.247.50): 56 data bytes
64 bytes from 192.168.247.50: seq=0 ttl=253 time=0.822 ms
64 bytes from 192.168.247.50: seq=1 ttl=253 time=0.879 ms
^C
--- 192.168.247.50 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.822/0.850/0.879 ms
```

## Troubleshooting CNI

### CNI not starting
```
kubectl get ds -n kube-system
NAME         DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
aws-node     3         3         3         3            3           <none>          8h
kube-proxy   3         3         3         3            3           <none>          8h
```

```
kubectl get pod -n kube-system
NAME                        READY     STATUS    RESTARTS   AGE
aws-node-2c5zn              1/1       Running   0          3h
aws-node-ng546              1/1       Running   0          3h
aws-node-wx4nh              1/1       Running   1          3h
kube-dns-64b69465b4-57l8d   3/3       Running   0          8h
kube-proxy-8mf7f            1/1       Running   0          3h
kube-proxy-9t9n8            1/1       Running   0          3h
kube-proxy-nmnz9            1/1       Running   0          3h
```

* EKS, security group given to (aws eks create-cluster...) is not right
* worker node have proxy setup
* worker node have taint

```
# find out kubernet SVC
kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   8h
```

```
#inside worker node
yum install -y telnet

# check if can reach k8s control plane
telnet 10.100.0.1 443
Trying 10.100.0.1...
Connected to 10.100.0.1.
Escape character is '^]'.
```

### key cni debug command

```
# /opt/cni/bin/aws-cni-support.sh
curl http://localhost:61678/v1/enis | python -m json.tool
 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   675  100   675    0     0    675      0  0:00:01 --:--:--  0:00:01  131k
{
    "AssignedIPs": 3,
    "ENIIPPools": {
        "eni-001197a0eeab79df8": {
            "AssignedIPv4Addresses": 1,
            "DeviceNumber": 0,
            "ID": "eni-001197a0eeab79df8",
            "IPv4Addresses": {
                "192.168.141.2": {
                    "Assigned": true
                },
                "192.168.145.92": {
                    "Assigned": false
                },
                "192.168.155.88": {
                    "Assigned": false
                },
                "192.168.160.34": {
                    "Assigned": false
                },
                "192.168.172.159": {
                    "Assigned": false
                }
            },
            "IsPrimary": true
        },
        "eni-0d7778f429dda454f": {
            "AssignedIPv4Addresses": 2,
            "DeviceNumber": 3,
            "ID": "eni-0d7778f429dda454f",
            "IPv4Addresses": {
                "192.168.130.206": {
                    "Assigned": true
                },
                "192.168.143.101": {
                    "Assigned": false
                },
                "192.168.157.205": {
                    "Assigned": true
                },
                "192.168.161.148": {
                    "Assigned": false
                },
                "192.168.164.163": {
                    "Assigned": false
                }
            },
            "IsPrimary": false
        }
    },
    "TotalIPs": 10
}
```

``` 
# cni debugging log file
cd /var/log/aws-routed-eni/

ls
ipamd.log.2018-07-17-22  ipamd.log.2018-07-17-23  ipamd.log.2018-07-18-00  ipamd.log.2018-07-18-01  plugin.log.2018-07-18-00  plugin.log.2018-07-18-01
```


## install calico policy add-on

```
kubectl apply -f calico.yaml 
daemonset.extensions "calico-node" created
customresourcedefinition.apiextensions.k8s.io "felixconfigurations.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "bgpconfigurations.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "ippools.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "hostendpoints.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "clusterinformations.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "globalnetworkpolicies.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "globalnetworksets.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "networkpolicies.crd.projectcalico.org" created
serviceaccount "calico-node" created
clusterrole.rbac.authorization.k8s.io "calico-node" created
clusterrolebinding.rbac.authorization.k8s.io "calico-node" created
deployment.extensions "calico-typha" created
clusterrolebinding.rbac.authorization.k8s.io "typha-cpha" created
clusterrole.rbac.authorization.k8s.io "typha-cpha" created
configmap "calico-typha-horizontal-autoscaler" created
deployment.extensions "calico-typha-horizontal-autoscaler" created
role.rbac.authorization.k8s.io "typha-cpha" created
serviceaccount "typha-cpha" created
rolebinding.rbac.authorization.k8s.io "typha-cpha" created
service "calico-typha" created
 
```
 
#### Examine calico add-on  

```
 kubectl get pod -n kube-system
NAME                                                  READY     STATUS    RESTARTS   AGE
aws-node-2c5zn                                        1/1       Running   0          3h
aws-node-ng546                                        1/1       Running   0          3h
aws-node-wx4nh                                        1/1       Running   1          3h
calico-node-g779n                                     1/1       Running   0          1m
calico-node-k2svs                                     1/1       Running   0          1m
calico-node-wmzbw                                     1/1       Running   0          1m
calico-typha-75667d89cb-7m4jr                         1/1       Running   0          1m
calico-typha-horizontal-autoscaler-78f747b679-qf965   1/1       Running   0          1m
kube-dns-64b69465b4-57l8d                             3/3       Running   0          8h
kube-proxy-8mf7f                                      1/1       Running   0          3h
kube-proxy-9t9n8                                      1/1       Running   0          3h
kube-proxy-nmnz9                                      1/1       Running   0          3h
```    

### [Simple Policy Demo](https://docs.projectcalico.org/v3.0/getting-started/kubernetes/tutorials/simple-policy)

#### Configure Namespaces
```
kubectl create ns policy-demo
```

#### Create demo pods

```
# Run the Pods.
kubectl run --namespace=policy-demo nginx --replicas=2 --image=nginx

# Create the Service.
kubectl expose --namespace=policy-demo deployment nginx --port=80
```

```
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.

/ # wget -q nginx -O -
```

```
# enable isolation
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: extensions/v1beta1
metadata:
  name: default-deny
  namespace: policy-demo
spec:
  podSelector:
    matchLabels: {}
EOF
```

```
# test isolation
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.

/ # wget -q --timeout=5 nginx -O -
wget: download timed out
/ #
```

```
Allow Access using a Network Policy
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: extensions/v1beta1
metadata:
  name: access-nginx
  namespace: policy-demo
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: access
EOF
```

```
# with label is able to access nginx
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.

/ # wget -q --timeout=5 nginx -O -
```

```
# Run a Pod without label and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo cant-access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/cant-access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.

/ # wget -q --timeout=5 nginx -O -
wget: download timed out
/ #
```

```
# cleanup
kubectl delete ns policy-demo
```




