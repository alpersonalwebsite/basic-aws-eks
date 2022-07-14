# Basic AWS EKS

## Pre req 

### Install AWS CLI
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Then run `aws configure`
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html

### Install eksctl
For the creation and management of EKS clusters
https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

### Install kubectl
https://kubernetes.io/docs/tasks/tools/


### Save user password in parameter store

Add user password to parameter store:

```shell
aws ssm put-parameter --name EKSUserPassword --type SecureString --value "Pass@1230765" --region us-west-1
```

Output:

```shell
{
    "Version": 1,
    "Tier": "Standard"
}
```

### Create EC2 Key pair

```shell
aws ec2 create-key-pair --key-name EKSProjectEC2KeyPair --region us-west-1  --output text > ~/.ssh/EKSProjectEC2KeyPair.pem
```
<!--
???
https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-keypairs.html
Setting permissions:

chmod 400 MyKeyPair.pem
-->


<!--
Deploy CloudFormation template
We are creating a user with CFN so the create an access key we need first the user
-->

### Create User Access Key

```shell
aws iam create-access-key --user-name user123 --region us-west-1
```

These will retrieve something like:

```shell
{
    "AccessKey": {
        "UserName": "user123",
        "AccessKeyId": "AKIAYQ5***",
        "Status": "Active",
        "SecretAccessKey": "qgm8s/1td1GXkOIvHYGm3CV***",
        "CreateDate": "2022-07-11T21:53:17+00:00"
    }
}
```

Save your `AccessKeyId` and `SecretAccessKey`


### Create EKS cluster
1 nodegroup and 3 worker nodes (t2.small)

We are going to use a customized configuration file for the cluster.

```shell
eksctl create cluster -f eksctl/cluster.yaml
```

This will deploy 2 CloudFormation templates provisioning all the resources you will need:
1. `eksctl-basic-eks-cluster-cluster`
2. `eksctl-basic-eks-cluster-nodegroup-ng-1`

If you want to use a mix of on-demand and spot instances

```
eksctl create cluster -f eksctl/cluster-demand-spot-instances.yaml
```

You can also add a new nodegroup to the cluster configuration file (previously used and create a new nodegroup) and execute:

```shell
eksctl create nodegroup --config-file=eksctl/cluster.yaml --include='ng-demand-spot-instances'
```


### Interacting with our cluster 

??? title

```shell
eksctl get nodegroup --cluster basic-eks-cluster --region us-west-1
```

Output:

```
CLUSTER			NODEGROUP	STATUS		CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID		ASG NAME
basic-eks-cluster	ng-1		CREATE_COMPLETE	2022-07-11T23:13:40Z	3		3		3			t2.small	ami-0f3ed62c68d8a3123	eksctl-basic-eks-cluster-nodegroup-ng-1-NodeGroup-D27KYK3RC1ZP
```

#### Scaling nodegroup
(If you want to scale up/down...)


```shell
eksctl scale nodegroup --cluster basic-eks-cluster --region us-west-1 \
  --nodes 4 \
  --nodes-max 4 \
  --name ng-1
```

Output:

```
2022-07-13 14:11:48 [ℹ]  scaling nodegroup "ng-1" in cluster basic-eks-cluster
2022-07-13 14:11:49 [ℹ]  nodegroup successfully scaled
```

If you execute again `eksctl get nodegroup --cluster basic-eks-cluster --region us-west-1` you will see:

```
CLUSTER			NODEGROUP	STATUS		CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID		ASG NAME
basic-eks-cluster	ng-1		CREATE_COMPLETE	2022-07-11T23:13:40Z	3		4		4			t2.small	ami-0f3ed62c68d8a3123	eksctl-basic-eks-cluster-nodegroup-ng-1-NodeGroup-D27KYK3RC1ZP
```

#### Adding and removing a nodegroup

**ADD**
Add a nodegroup to the cluster configuration file and then execute:

```shell
eksctl create nodegroup --config-file=eksctl/cluster.yaml --include='ng-new-nodegroup'
```

**REMOVE**

```shell
eksctl delete nodegroup --config-file=eksctl/cluster.yaml --include='ng-new-nodegroup' --approve 
```


## Clean up

### Delete user password from parameter store

```shell
aws ssm delete-parameter --name EKSUserPassword --region us-west-1
```

### Delete the EC2 Key Pair

```shell
aws ec2 delete-key-pair --key-name EKSProjectEC2KeyPair --region us-west-1
```