# KOPS Demo

## Objective

### Create kops cluster

```
- In public subnet
- One master node
- Two worker nodes across two availability zones
```

### Deploy application across the cluster

```
- Deploy busybox application
- Deploy hello-world application
```

### Packet flow for 3 communications pattern:

```
- pod-to-pod communication
- pod-to-service communication
- external-to-interal communication
```

### Understand concepts and limitations:

```
- Pods
- Service (types: Cluster-IP, NodePort, LoadBalance, Ingress)
- Network architecture (interfaces, bridge, route table, ip address)
```

## Getting Started

### Create kops cluster using [AWS CloudFormation Template](https://aws.amazon.com/cloudformation/):

```
- **For this workshop activity, we are using AWS CloudFormation template to configure "Getting Started".**

- If you have not launch the template or template did not launch successfully, you can launch AWS CloudFormation template from link below, preferrably in eu-west-1 region:

  - [CloudFormation Template: NET410 Workshop Setup](https://s3-eu-west-1.amazonaws.com/net410-workshop-eu-west-1/net410-workshop-setup.json)

```

### Setup environment:

```
- Use OS/machine of your choice.
- In this guide we are going to create a dedicated amazon ec2 instance and install necessary tools
  - Refer to appendix A for how to create a instance for cluster operations
```

#### Create ssh key-pair to access amazon ec2 instnace:

#### Launch amazon ec2 instance (kops-setup-instance):

#### Access kops-setup-instance:

```
ssh -i <path_to_your_ssh_private_kye.pem> ec2-user@<ec2-instance-public-ip>
```

#### Configure kops setup instance for AWS access:

- Configure AWS CLI:
```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```


#### IAM user permission:

- In order to create cluster with AWS, IAM user requires following permissions:
  - AmazonEC2FullAccess
  - AmazonS3FullAccess
  - IAMFullAccess
  - AmazonVPCFullAccess
  - AmazonRoute53FullAccess

- You can add permissions to your existing users or you can create a new user.
  - For this activity, we will create a new IAM user: 'kops'

```
aws iam create-group --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
aws iam create-access-key --user-name kops

```

#### DNS Setup:

- Configure DNS to suite your needs.
  - Kops 1.6.2 or later allows you to create a gossip-based cluster.
  - If you are using gossip-based cluster, you don't need to configure DNS.
  - When using gossip-based cluster, your cluster name should end with .k8s.local

- For this activity we are going to create gossip-based cluster.
  - Refer to Appendix B for DNS configuaration when using non gossip-based cluster

#### Cluster State Storage:

- Kops requires to store the state and represenation of the cluster.
  - Create a dedicated amazon s3 bucket that kops can use for this purpose

- Note:
  - S3 requires --create-bucket-configuration LocationConstraint=<region> for regions other than us-east-1.
  - It is STRONGLY recommend to enable versioning on your S3 bucket. Versioning allows you to revert or recover a previous state store.

```
aws s3api create-bucket --bucket net410-kops-cluster-state-store-<aws-account-id> --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
aws s3api put-bucket-versioning --bucket net410-kops-cluster-state-store-<aws-account-id>  --versioning-configuration Status=Enabled
```

- export KOPS_STATE_STORE:
  - Kops will use this location by default.
  - It is recommended to putting this in your bash profile or similar.

```
KOPS_STATE_STORE=s3://net410-kops-cluster-state-store-<aws-account-id>
```

#### Create SSH key-pair:

- Create new or use existing ssh key-pair:
  - For this activity, we will create SSH key-pair to access the cluster. You can use your own key-pair.

```
aws ec2 create-key-pair --key-name net410_clusterAdmin |jq -r '.KeyMaterial’ >net410_admin.pem
ssh-keygen -y -f net410_admin.pem >net410_clusterAdmin.pub
```

#### Create kops cluster:

- Create cluster using kops command line utility:

```
kops create cluster \
  --name net410-kops-cluster.k8s.local \
  --master-count 1 \
  --master-size "t2.small" \
  --node-count 1 \
  --node-size "t2.small" \
  --network-cidr "10.0.0.0/16" \
  --zones "us-west-2a,us-west-2b,us-west-2c" \
  --ssh-public-key "~/.ssh/k8s_admin.pub" \
  --networking "kubenet"
```

- Create command only creates configuraiton for the cluster. It does not create any AWS cloud resources
  - This allows you to review your cluster configure before creating resources, use following commands to review/edit your cluster:

```
kops get cluster
kops edit cluster net410-kops-cluster.k8s.local
kops edit ig --name net410-kops-cluster.k8s.local nodes
kops edit ig --name net410-kops-cluster.k8s.local master-us-west-2a
```

- Create AWS cloud resources after reviewing cluster configuration

```
kops update net410-kops-cluster.k8s.local --yes
```

## Deploy busybox application on the cluster

### Cluster information:

- Cluster that was created using [AWS CloudFormation](https://aws.amazon.com/cloudformation/)consists of 1 master node and 2 worker nodes.

```
$: kops get cluster
NAME                CLOUD   ZONES
net410-kops-cluster.k8s.local   aws us-west-2a,us-west-2b,us-west-2c
$: kops validate cluster
Using cluster from kubectl context: net410-kops-cluster.k8s.local

Validating cluster net410-kops-cluster.k8s.local

INSTANCE GROUPS
NAME            ROLE    MACHINETYPE MIN MAX SUBNETS
master-us-west-2a   Master  t2.small    1   1   us-west-2a
nodes           Node    t2.small    2   2   us-west-2a,us-west-2b,us-west-2c

NODE STATUS
NAME                        ROLE    READY
ip-10-1-1-28.us-west-2.compute.internal     master  True
ip-10-1-2-179.us-west-2.compute.internal    node    True
ip-10-1-3-165.us-west-2.compute.internal    node    True

Your cluster net410-kops-cluster.k8s.local is ready
$:
```

- Cluster node information:

```
$: kubectl get nodes -o wide
NAME                                       STATUS   ROLES    AGE   VERSION   EXTERNAL-IP      OS-IMAGE                      KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-1-1-28.us-west-2.compute.internal    Ready    master   2d    v1.10.6   34.218.224.163   Debian GNU/Linux 8 (jessie)   4.4.121-k8s      docker://17.3.2
ip-10-1-2-179.us-west-2.compute.internal   Ready    node     2d    v1.10.6   18.236.63.230    Debian GNU/Linux 8 (jessie)   4.4.121-k8s      docker://17.3.2
ip-10-1-3-165.us-west-2.compute.internal   Ready    node     2d    v1.10.6   18.237.189.223   Debian GNU/Linux 8 (jessie)   4.4.121-k8s      docker://17.3.2
$:
```

- For cluster operations, it did create pods in kube-system namespace:

```
$: kubectl get pods -o wide -n kube-system
NAME                                                              READY   STATUS    RESTARTS   AGE   IP             NODE
dns-controller-6d6b7f78b-5cznb                                    1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
etcd-server-events-ip-10-1-1-28.us-west-2.compute.internal        1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
etcd-server-ip-10-1-1-28.us-west-2.compute.internal               1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-apiserver-ip-10-1-1-28.us-west-2.compute.internal            1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-controller-manager-ip-10-1-1-28.us-west-2.compute.internal   1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-dns-5fbcb4d67b-hr4vr                                         3/3     Running   0          17h   100.65.129.4   ip-10-1-2-179.us-west-2.compute.internal
kube-dns-5fbcb4d67b-s26x8                                         3/3     Running   0          2d    100.65.130.2   ip-10-1-3-165.us-west-2.compute.internal
kube-dns-autoscaler-6874c546dd-k2twt                              1/1     Running   0          2d    100.65.129.2   ip-10-1-2-179.us-west-2.compute.internal
kube-proxy-ip-10-1-1-28.us-west-2.compute.internal                1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-proxy-ip-10-1-2-179.us-west-2.compute.internal               1/1     Running   0          2d    10.1.2.179     ip-10-1-2-179.us-west-2.compute.internal
kube-proxy-ip-10-1-3-165.us-west-2.compute.internal               1/1     Running   0          2d    10.1.3.165     ip-10-1-3-165.us-west-2.compute.internal
kube-scheduler-ip-10-1-1-28.us-west-2.compute.internal            1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kubernetes-dashboard-7b9c7bc8c9-ttc7z                             1/1     Running   0          2d    100.65.129.3   ip-10-1-2-179.us-west-2.compute.internal
$:
```

- Application is not deployed yet, hence, you won't see any pods running in default name space:

```
$: kubectl get pods -o wide
No resources found.
$: kubectl get pods -n default -o wide
No resources found.
$:
```

- View pods in all namepsace by specifying --all-namespaces:

```
$: kubectl get pods -o wide --all-namespaces
NAMESPACE     NAME                                                              READY   STATUS    RESTARTS   AGE   IP             NODE
kube-system   dns-controller-6d6b7f78b-5cznb                                    1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   etcd-server-events-ip-10-1-1-28.us-west-2.compute.internal        1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   etcd-server-ip-10-1-1-28.us-west-2.compute.internal               1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   kube-apiserver-ip-10-1-1-28.us-west-2.compute.internal            1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   kube-controller-manager-ip-10-1-1-28.us-west-2.compute.internal   1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   kube-dns-5fbcb4d67b-hr4vr                                         3/3     Running   0          18h   100.65.129.4   ip-10-1-2-179.us-west-2.compute.internal
kube-system   kube-dns-5fbcb4d67b-s26x8                                         3/3     Running   0          2d    100.65.130.2   ip-10-1-3-165.us-west-2.compute.internal
kube-system   kube-dns-autoscaler-6874c546dd-k2twt                              1/1     Running   0          2d    100.65.129.2   ip-10-1-2-179.us-west-2.compute.internal
kube-system   kube-proxy-ip-10-1-1-28.us-west-2.compute.internal                1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   kube-proxy-ip-10-1-2-179.us-west-2.compute.internal               1/1     Running   0          2d    10.1.2.179     ip-10-1-2-179.us-west-2.compute.internal
kube-system   kube-proxy-ip-10-1-3-165.us-west-2.compute.internal               1/1     Running   0          2d    10.1.3.165     ip-10-1-3-165.us-west-2.compute.internal
kube-system   kube-scheduler-ip-10-1-1-28.us-west-2.compute.internal            1/1     Running   0          2d    10.1.1.28      ip-10-1-1-28.us-west-2.compute.internal
kube-system   kubernetes-dashboard-7b9c7bc8c9-ttc7z                             1/1     Running   0          2d    100.65.129.3   ip-10-1-2-179.us-west-2.compute.internal
$:

```

- Worker node networking details:
  - access one of the work node using ssh key-pair

```
admin@ip-10-1-2-179:~$ ip link show |grep -i "state up"
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
4: cbr0: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 9001 qdisc htb state UP mode DEFAULT group default qlen 1000
5: veth7fcda148@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue master cbr0 state UP mode DEFAULT group default
6: veth607c472d@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue master cbr0 state UP mode DEFAULT group default
7: vethff861804@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue master cbr0 state UP mode DEFAULT group default
admin@ip-10-1-2-179:~$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 06:97:df:a2:0f:f4 brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.179/24 brd 10.1.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::497:dfff:fea2:ff4/64 scope link
       valid_lft forever preferred_lft forever
admin@ip-10-1-2-179:~$

admin@ip-10-1-2-179:~$ ip addr show cbr0
4: cbr0: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 9001 qdisc htb state UP group default qlen 1000
    link/ether 0a:58:64:41:81:01 brd ff:ff:ff:ff:ff:ff
    inet 100.65.129.1/24 scope global cbr0
       valid_lft forever preferred_lft forever
    inet6 fe80::dced:1dff:fe3d:6f03/64 scope link
       valid_lft forever preferred_lft forever
admin@ip-10-1-2-179:~$ sudo brctl show
bridge name bridge id       STP enabled interfaces
cbr0        8000.0a5864418101   no      veth607c472d
                                        veth7fcda148
                                        vethff861804
docker0     8000.02424e9c2262   no
admin@ip-10-1-2-179:~$ ip route show
default via 10.1.2.1 dev eth0
10.1.2.0/24 dev eth0  proto kernel  scope link  src 10.1.2.179
100.65.129.0/24 dev cbr0  proto kernel  scope link  src 100.65.129.1
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1
admin@ip-10-1-2-179:~$
```

- vethxx interfaces are for pods that are running kube-system name space:

```
$: kubectl get pods -o wide --all-namespaces |grep 10-1-2-179
kube-system   kube-dns-5fbcb4d67b-hr4vr                                         3/3     Running   0          18h   100.65.129.4   ip-10-1-2-179.us-west-2.compute.internal
kube-system   kube-dns-autoscaler-6874c546dd-k2twt                              1/1     Running   0          2d    100.65.129.2   ip-10-1-2-179.us-west-2.compute.internal
kube-system   kube-proxy-ip-10-1-2-179.us-west-2.compute.internal               1/1     Running   0          2d    10.1.2.179     ip-10-1-2-179.us-west-2.compute.internal
kube-system   kubernetes-dashboard-7b9c7bc8c9-ttc7z                             1/1     Running   0          2d    100.65.129.3   ip-10-1-2-179.us-west-2.compute.internal
$:
```

### Deploy application

- Application be deployed:
  1. By directly passing arguments to kubectl cli
  2. Define resource in a file (yaml or jason format; yaml preferred) and pass file as an argument to kubectl cli.
     - Advantage of using file base approach is it allows to keep track of what's launched and allows to make any changes.

- For this workshop will deploy it using a configuraiton file.
  - Configuration files are located under $HOME/kops/

```
$: kubectl create -f busyboxDeployment.yaml
deployment.apps/net410-kops-busybox created
$:
```

- It will create two pods, one on each node:

```
$: kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP             NODE
net410-kops-busybox-55d8557b4d-6gxrj   1/1     Running   0          1m    100.65.130.9   ip-10-1-3-165.us-west-2.compute.internal
net410-kops-busybox-55d8557b4d-b8w5n   1/1     Running   0          1m    100.65.129.6   ip-10-1-2-179.us-west-2.compute.internal
$:
```

- Lets check the node networking details again:

  - You should see vethxx for busybox pod:

```
admin@ip-10-1-2-179:~$ sudo brctl show cbr0
bridge name bridge id       STP enabled interfaces
cbr0        8000.0a5864418101   no      veth607c472d
                                        veth7fcda148
                                        veth8175afaa
                                        vethff861804
admin@ip-10-1-2-179:~$

admin@ip-10-1-2-179:~$ ip addr show veth8175afaa
9: veth8175afaa@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue master cbr0 state UP group default
    link/ether 02:62:8f:a3:5d:44 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::62:8fff:fea3:5d44/64 scope link
       valid_lft forever preferred_lft forever
admin@ip-10-1-2-179:~$
```

- Access the pod:
