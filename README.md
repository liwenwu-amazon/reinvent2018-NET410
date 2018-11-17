# AWS re:Invent 2018: NET410 Workshop

## Workshop Environment:

### On Linux create ssh key-pair

- Using [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-ec2-keypairs.html):
```
$: aws --region eu-west-1 ec2 create-key-pair --key-name mySshKey --query 'KeyMaterial' --output text > ~/.ssh/mySshKey.pem
$: chmod 400 ~/.ssh/mySshKey.pem
$: ssh-keygen -y -f ~/.ssh/mySshKey.pem > ~/.ssh/mySshKey.pub
```

- Using ssh-keygen utility **Leave the passphrase empty**:
```
$: ssh-keygen -f ~/.ssh/mySshKey -t rsa
$: cp ~/.ssh/mySshKey ~/.ssh/mySshKey.pem
$: aws iam upload-ssh-public-key --user-name <iam-user-name> --ssh-public-key-body 'public key string'
```

### Create two clusters:

- Right click on [AWS CloudFormation Template](https://s3-eu-west-1.amazonaws.com/net410-workshop-eu-west-1/net410-workshop-setup.json) and save the CloudFormation template (click on save link as) on your desktop
- Open [AWS CloudFormation Console in eu-west-1 (Ireland) region](https://eu-west-1.console.aws.amazon.com/cloudformation/) and launch the CloudFomration template
- Launch CloudFormation template with default values except of ssh key-pair. Please use existing or newly created ssh key-pair

- AWS CloudFormation Template creates:
  - Two Amazon EC2 instances for workshop setup:
    1. net410-workshop -kops
    2. net410-workshop -eks
  - Setup up environment including install necessary tools
  - Creates two clusters:
    1. [kops - kubernetes sperations](https://github.com/kubernetes/kops/blob/master/README.md)
    2. [Amazon EKS Cluster](https://aws.amazon.com/eks/)


