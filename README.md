# AWS re:Invent 2018: NET410 Workshop

## Workshop Environment:

- Presentation slides available on
[slideshare](https://www.slideshare.net/AmazonWebServices/deep-dive-on-container-networking-at-scale-on-amazon-eks-amazon-ecs-amazon-ec2-net410-aws-reinvent-2018?qid=6daf31f7-0051-4024-9ab9-d81b1f2fe614&v=&b=&from_search=1)

### Create ssh key-pair:

- **Skip this step** if you want to use your existing ssh key-pair

#### Using Amazon EC2 console

- To create ssh key-pair using Amazon EC2 Console, click on [Amazon EC2 console](https://eu-west-1.console.aws.amazon.com/ec2/).
  - Documentation can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair)
  - Verify region is set to eu-west-1 (Ireland)

### Create two clusters:

- **For this workshop, we will use AWS CloudFormation to create two clusters**
- Launch [AWS CloudFormation Template in eu-west-1 (Ireland) region](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?templateURL=https://s3-eu-west-1.amazonaws.com/net410-workshop-eu-west-1/net410-workshop-setup.json)
  - Launch CloudFormation template with **default values except of ssh key-pair**. Please use existing or newly created ssh key-pair
- Workshop activity is restricted to **eu-west-1 (Ireland) region**. Trying to launch AWS CloudFormation template in any other region will fail.

- AWS CloudFormation Template creates:
  - Two Amazon EC2 instances for workshop setup and management:
    1. **net410-workshop-kops-mgmt**
    2. **net410-workshop-eks-mgmt**
  - Sets up environment including installing necessary tools
  - Creates two clusters:
    1. [kops - kubernetes sperations](https://github.com/kubernetes/kops/blob/master/README.md)
        - 1 master node
        - 2 worker nodes across 2 availability zones
    2. [Amazon EKS Cluster](https://aws.amazon.com/eks/)
        - Control plane is managed by Amazon EKS
        - 2 worker nodes across 2 availability zones

- **AWS CloudFormation will fail if default VPC does not exist**. Make sure you have default VPC
    - Issue will be fixed soon. Until then please make sure you have default VPC

### CloudFormation Template:

- [NET410 Workshop Setup: CloudFormation Template](https://s3-eu-west-1.amazonaws.com/net410-workshop-eu-west-1/net410-workshop-setup.json)
