# AWS re:Invent 2018: NET410 Workshop

## Workshop Environment:

### Create two clusters:

- Right click on [AWS CloudFormation Template](https://s3-eu-west-1.amazonaws.com/net410-workshop-eu-west-1/net410-workshop-setup.json) and save the CloudFormation template (click on save link as) on your desktop
- Open [AWS CloudFormation Console in eu-west-1 (Ireland) region](https://eu-west-1.console.aws.amazon.com/cloudformation/) and launch the CloudFomration template

- AWS CloudFormation Template creates:
  - Two Amazon EC2 instances for workshop setup:
    1. net410-workshop -kops
    2. net410-workshop -eks
  - Setup up environment including install necessary tools
  - Creates two clusters:
    1. [kops - kubernetes sperations](https://github.com/kubernetes/kops/blob/master/README.md)
    2. [Amazon EKS Cluster](https://aws.amazon.com/eks/)


