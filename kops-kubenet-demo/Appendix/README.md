## Getting Started:

- Use OS/machine of your choice.
- In this guide we are going to create a dedicated amazon ec2 instance and install necessary tool

### On Linux install and configure [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/awscli-install-linux.html):

- Install:

```
$: pip install awscli --upgrade --user
```

- Configure:
```
$: aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```

### On Linux create ssh key-pair:

- ssh key-pair is used to access amazon ec2 instance:

```
$: aws --region eu-west-1 ec2 create-key-pair --key-name mySshKey --query 'KeyMaterial' --output text > ~/.ssh/mySshKey.pem
$: chmod 400 ~/.ssh/mySshKey.pem
$: ssh-keygen -y -f ~/.ssh/mySshKey.pem > ~/.ssh/mySshKey.pub
```

### Launch Amazon EC2 Linux instance:

- Launch it using ec2 console or aws cli:
  - example of launching it using aws cli:

```
$: aws ec2 run-instances --image-id <Amazon Linux 2 AMI id> --count 1 --instance-type t2.micro --key-name ~/.ssh/mySshKey.pub --security-group-ids <security-group-id> --subnet-id <subnet-id>
```

### Copy ssh key-pair to Amazon EC2 instance:

- We will use the same key pair to create kops cluster
```
$: scp ~/.ssh/mySshKey.* ec2-user@<ec2-instance-public-ip>
```
### Access Amazon EC2 instance:

```
$: ssh -i ~/.ssh/mySshKey.pem ec2-user@<ec2-instance-public-ip>
```

### Install kubectl:

```
$ wget -O kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
$ chmod +x kubectl
```

### Install kops:

```
$ wget -O kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
$ chmod +x kops
```

### Install jq, lightweight and flexible command-line JSON processor:

```
$ yum install jq -y
```

#### Configure kops setup instance for AWS access:

- Configure aws cli on the Amazon EC2 instance:
```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```

#### IAM user permission:

- To create kops cluster in AWS, IAM user requires following permissions:
  - AmazonEC2FullAccess
  - AmazonS3FullAccess
  - IAMFullAccess
  - AmazonVPCFullAccess
  - AmazonRoute53FullAccess

- You can add permissions to your existing users or you can create a new user.
  - For this activity, we will create a new IAM user: 'kops'
```
$ aws iam create-group --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
$ aws iam create-user --user-name kops
$ aws iam add-user-to-group --user-name kops --group-name kops
$ aws iam create-access-key --user-name kops
```

#### DNS Setup:

- Configure DNS to suite your needs.
  - Kops 1.6.2 or later allows you to create a gossip-based cluster.
  - If you are using gossip-based cluster, you don't need to configure DNS.
  - When using gossip-based cluster, your cluster name should end with .k8s.local

- For this activity we are going to create gossip-based cluster.

#### Cluster State Storage:

- Kops requires to store the state and represenation of the cluster.
  - Create a dedicated amazon s3 bucket that kops can use for this purpose

- Note:
  - S3 requires --create-bucket-configuration LocationConstraint=<region> for regions other than us-east-1.
  - It is STRONGLY recommend to enable versioning on your S3 bucket. Versioning allows you to revert or recover a previous state store.
```
$ aws s3api create-bucket --bucket net410-kops-cluster-state-store-<aws-account-id> --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
$ aws s3api put-bucket-versioning --bucket net410-kops-cluster-state-store-<aws-account-id>  --versioning-configuration Status=Enabled
```

- export KOPS_STATE_STORE:
  - Kops will use this location by default.
  - It is recommended to save KOPS_STATE_STORE in .bash_profile or similar.
```
$ export KOPS_STATE_STORE="s3://net410-kops-cluster-state-${AWS_ACCOUNT_ID}-${AWS_DEFAULT_REGION}"
```

#### Create kops cluster:

- Create cluster using kops command line utility:
```
$ kops create cluster \
  --name net410-kops-cluster.k8s.local \
  --master-count 1 \
  --master-size "t2.small" \
  --node-count 1 \
  --node-size "t2.small" \
  --network-cidr "10.0.0.0/16" \
  --zones "us-west-2a,us-west-2b,us-west-2c" \
  --ssh-public-key "~/.ssh/mySshKey.pub" \
  --networking "kubenet"
```

- Create command only creates configuraiton for the cluster. It does not create any AWS cloud resources
  - This allows you to review your cluster configure before creating resources, use following commands to review/edit your cluster:
```
$ kops get cluster
$ kops edit cluster net410-kops-cluster.k8s.local
$ kops edit ig --name net410-kops-cluster.k8s.local nodes
$ kops edit ig --name net410-kops-cluster.k8s.local master-us-west-2a
```

- Create AWS cloud resources after reviewing cluster configuration
```
$ kops update net410-kops-cluster.k8s.local --yes
```

- Use following commands to validate cluster creation:
```
$ kubeclt cluster-info
$ kops get cluster
$ kops validate cluster
```
