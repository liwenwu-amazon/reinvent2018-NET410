# AWS VPC routed CNI and Amazon EKS Cluster Networking

## Objective:

* Exam major components of AWS VPC routed CNI and demonstrace how it handles traffic for:
	* Pod to Pod communication
	* Pod to Service communication
* Exam the communication channels between Amazon EKS Kubernetes Control Plane and Customer worker nodes.

## Workshop setup:

### [AWS CloudFormation Template](https://aws.amazon.com/cloudformation/):

- **For this workshop activity, we are using AWS CloudFormation template to configure workshop setup.**
- You should have launched the template at the beginning of the session and your cluster should already be up and running
- If you did not launch the template or template did not launch successfully, you can re-launch AWS CloudFormation template from link below. **Launch it in eu-west-1 (Ireland) region**:

  - [CloudFormation Template: NET410 Workshop Setup](https://s3-eu-west-1.amazonaws.com/net410-workshop-eu-west-1/net410-workshop-setup.json)

### ssh into EKS console

* Find out `EksEC2Instance` and EC2 instance ID from Cloudformation resource
* ssh into EKS instance

```
ssh -i <key-name>  ec2-user@ec2-xx-xx-xx-xx.eu-west-1.compute.amazonaws.com

```

### Clone github:

- Before we begin, clone reinvent2018-NET410 github repository in $HOME directory:
```
[ec2-user@ip-172-31-25-39 ~]$ git clone https://github.com/liwenwu-amazon/reinvent2018-NET410
```

## NET410 workshop activity: EKS cluster

## Amazon EKS Cluster

### Cluster Information

- Cluster that was created using [AWS CloudFormation](https://aws.amazon.com/cloudformation/) consists of 1 master node and 2 worker nodes.

#### aws eks list-clusters
```
aws eks list-clusters
{
    "clusters": [
        "net410-eks-cluster"
    ]
}
```

#### aws eks describe-cluster --name net410-eks-cluster

```
aws eks describe-cluster --name net410-eks-cluster
{
    "cluster": {
        "status": "ACTIVE", 
        "endpoint": "https://C23A88F2572AAF0B1AEA36CD119D0682.yl4.eu-west-1.eks.amazonaws.com", 
        "name": "net410-eks-cluster", 
        "certificateAuthority": {
            "data": "......   "
        }, 
        "roleArn": "arn:aws:iam::694065802095:role/eksctl-net410-eks-cluster-cluster-ServiceRole-JQP22M9HN457", 
        "resourcesVpcConfig": {
            "subnetIds": [
                "subnet-03570d333db01c922", 
                "subnet-02b0202c2159049f3", 
                "subnet-04dfca08b2fc54441", 
                "subnet-0ff53ecf4d71f5a60", 
                "subnet-09d6c50e781c0e074", 
                "subnet-01b4a2c179e95f519"
            ], 
            "vpcId": "vpc-09c7c672286bf7e94", 
            "securityGroupIds": [
                "sg-062a376f3d16fe673"
            ]
        }, 
        "version": "1.10", 
        "arn": "arn:aws:eks:eu-west-1:xxxxx:cluster/net410-eks-cluster", 
        "createdAt": 1542590239.656
    }
}
```

#### Kubernetes Cluster Info

```
kubectl cluster-info
Kubernetes master is running at https://C23A88F2572AAF0B1AEA36CD119D0682.yl4.eu-west-1.eks.amazonaws.com

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```
[ec2-user@ip-172-31-9-36 ~]$ kubectl get node -o wide
NAME                                           STATUS   ROLES    AGE   VERSION   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION               CONTAINER-RUNTIME
ip-192-168-4-93.eu-west-1.compute.internal     Ready    <none>   15h   v1.10.3   <none>        Amazon Linux 2   4.14.72-73.55.amzn2.x86_64   docker://17.6.2
ip-192-168-77-144.eu-west-1.compute.internal   Ready    <none>   15h   v1.10.3   <none>        Amazon Linux 2   4.14.72-73.55.amzn2.x86_64   docker://17.6.2
```

### Amazon EKS Architecture

![Amazon EKS Architecture](./images/eks-arch.png)

### Communication between EKS Master and Pods (e.g. kube-dns)

![](./images/cross-eni.png)

* kubectl logs/exec uses this channel (EKS Master -> Pods) 
* Pods (e.g. kube-dns, aws-cni) use this channel to Read/Write/Watch Kubernetes API objects

```

aws ec2 describe-network-interfaces --network-interface-ids eni-0f96810f42e4f53e8 --region eu-west-1
{
    "NetworkInterfaces": [
        {
            "Attachment": {
                "AttachTime": "2018-11-19T01:27:39.000Z",
                "AttachmentId": "eni-attach-0b3be060aa470e238",
                "DeleteOnTermination": true,
                "DeviceIndex": 1,
                "InstanceOwnerId": "298800981437",
                "Status": "attached"
            },
            "AvailabilityZone": "eu-west-1a",
            "Description": "Amazon EKS net410-eks-cluster", <-- !! Cross Account ENI
            "Groups": [
                {
                    "GroupName": "eksctl-net410-eks-cluster-cluster-ControlPlaneSecurityGroup-XLHWMBXCDLTP",
                    "GroupId": "sg-062a376f3d16fe673"
                }
            ],
            "InterfaceType": "interface",
            "Ipv6Addresses": [],
            "MacAddress": "0a:d5:79:a1:af:50",
            "NetworkInterfaceId": "eni-0f96810f42e4f53e8",
            "OwnerId": "xxxx",
            "PrivateDnsName": "ip-192-168-154-135.eu-west-1.compute.internal",
            "PrivateIpAddress": "192.168.154.135",
            "PrivateIpAddresses": [
                {
                    "Primary": true,
                    "PrivateDnsName": "ip-192-168-154-135.eu-west-1.compute.internal",
                    "PrivateIpAddress": "192.168.154.135"
                }
            ],
            "RequesterId": "AROAIORXPPLFOF7XU4KUQ:AmazonEKS",
            "RequesterManaged": false,
            "SourceDestCheck": true,
            "Status": "in-use",
            "SubnetId": "subnet-09d6c50e781c0e074",
            "TagSet": [],
            "VpcId": "vpc-09c7c672286bf7e94"
        }
    ]
}
``` 
##### Troubleshooting Tips

* Misconfigured control plane security group
	* Control plane security group is assigned to ENIs created in the worker node subnets.
	* When launching worker nodes, control plane security group is configured to receive packets from worker nodes.
	* if different control plane security group is specified while creating worker nodes, pods will not be able to communicate with master
* VPC related issues 
	* Deleting subnets in your VPC
	* Removing Ingress and Egress required for Master and Worker node communication.
	* Reaching ENI limits for an AWS Account.
	* Exhausting IPs available for Cross ENI

* Incorrect permissions on the role could stop AmazonEKS from managing Kubernetes clusters.
	* Use Managed policy provided by EKS.
	* Avoid attaching deny permissions on APIs required by AmzonEKS for managing ENIs in your VPC.

	 
       


### Pod Netowrk Stack Internal

```
kubectl get node
NAME                              STATUS    ROLES     AGE       VERSION
ip-192-168-112-98.ec2.internal    Ready     <none>    14m       v1.10.3
ip-192-168-167-201.ec2.internal   Ready     <none>    14m       v1.10.3
ip-192-168-242-227.ec2.internal   Ready     <none>    14m       v1.10.3
```
### Create Pods

```
git clone https://github.com/liwenwu-amazon/reinvent2018-NET410

```

```
cd reinvent2018-NET410/eks-cni-demo
```

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





