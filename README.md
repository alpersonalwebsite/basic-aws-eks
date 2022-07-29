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

**For EKSUser**

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

**For EKSAdminUser**

```shell
aws iam create-access-key --user-name eks-admin-user --region us-west-1
```

```shell
{
    "AccessKey": {
        "UserName": "eks-admin-user",
        "AccessKeyId": "AKIAYQ5AHLCL3GX5P3BE",
        "Status": "Active",
        "SecretAccessKey": "6niSt4wbxr45rDGhHnndyxIIPsokL/KEgTceNf3A",
        "CreateDate": "2022-07-26T16:12:48+00:00"
    }
}
```

Add a new `profile` to `/Users/your-user/.aws/credentials`

```
[eks-admin]
aws_access_key_id = AKIAYQ5AHLCL3GX5P3BE
aws_secret_access_key = 6niSt4wbxr45rDGhHnndyxIIPsokL/KEgTceNf3A
region=us-west-1
```

**For EKSRegularUser**

```shell
aws iam create-access-key --user-name eks-user --region us-west-1
```

```shell
{
    "AccessKey": {
        "UserName": "eks-user",
        "AccessKeyId": "AKIAYQ5AHLCLWTL4DAXJ",
        "Status": "Active",
        "SecretAccessKey": "RwNRm56vlpQK+7LC4ftqSNkIf1Wy5ePDHySOcsNm",
        "CreateDate": "2022-07-26T19:48:36+00:00"
    }
}
```

Add a new `profile` to `/Users/your-user/.aws/credentials`

```
[eks-user]
aws_access_key_id = AKIAYQ5AHLCLWTL4DAXJ
aws_secret_access_key = RwNRm56vlpQK+7LC4ftqSNkIf1Wy5ePDHySOcsNm
region=us-west-1
```


### Create EKS cluster
1 nodegroup and 3 worker nodes (t2.small)

We are going to use a customized configuration file for the cluster.

```shell
eksctl create cluster -f eks/cluster.yaml

# If you want to delete your cluster: eksctl delete cluster --name basic-eks-cluster --region us-west-1
```

This will deploy 2 CloudFormation templates provisioning all the resources you will need:
1. `eksctl-basic-eks-cluster-cluster`
2. `eksctl-basic-eks-cluster-nodegroup-ng-1`

If you want to use a mix of on-demand and spot instances

```
eksctl create cluster -f eks/cluster-demand-spot-instances.yaml
```

You can also add a new nodegroup to the cluster configuration file (previously used and create a new nodegroup) and execute:

```shell
eksctl create nodegroup --config-file=eks/cluster.yaml --include='ng-demand-spot-instances'
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
(to scale up/down the cluster)

<!--
For stateful workloads we use a nodegroup within a single AZ because the EBS volumes cannot be shared across AZ
For stateless workloads we use a nodegroup within multiples AZ
-->

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
eksctl create nodegroup --config-file=eks/cluster.yaml --include='ng-new-nodegroup'
```

**REMOVE**

```shell
eksctl delete nodegroup --config-file=eks/cluster.yaml --include='ng-new-nodegroup' --approve 
```

#### AutoScaler

```shell
 eksctl create nodegroup --config-file=eks/cluster-autoscaling.yaml
```

<!--
What the following commands do?
-->

##### Create deployment for AutoScaler

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

Sample output:

```shell
serviceaccount/cluster-autoscaler created
clusterrole.rbac.authorization.k8s.io/cluster-autoscaler created
role.rbac.authorization.k8s.io/cluster-autoscaler created
clusterrolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
rolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
deployment.apps/cluster-autoscaler created
```

**Set annotation for deployment**
(to avoid being evicted)

```shell
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false" --overwrite
```

Sample output:

```shell
deployment.apps/cluster-autoscaler annotated
```

Now we check the version of EKS: https://us-west-1.console.aws.amazon.com/eks/home?region=us-west-1#/clusters
Example: `1.20`

Then we search for `1.20.x` release on https://github.com/kubernetes/autoscaler/releases
Example: `1.20.3`

**Configure cluster name and image version**

Now we edit `deployment.apps/cluster-autoscaler`

```shell
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

Then we search for `- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.iocluster-autoscaler/<YOUR CLUSTER NAME>` and...
1. We put the name of our cluster (basic-eks-cluster)
2. Switch to the proper image (1.20.3)

We can check our changes...

```shell
kubectl -n kube-system describe deployment cluster-autoscaler
```

<!-- 
  Check all commands, particularly kubectl and add explanations
-->

##### Deploy NGINX 

<!-- 
  It could be any container, right?
-->

```shell
kubectl apply -f eks/nginx-deployment.yaml
```

Output:

```
deployment.apps/testing-autoscaler created
```

Check that the pod is running

```shell
kubectl get pod
```

Sample output:

```
NAME                                  READY   STATUS    RESTARTS   AGE
test-autoscaler-7db49f8dc5-ghpdr      1/1     Running   0          92s
```

Now we can increase (or decrease) the number of replicas:

```shell
kubectl scale --replicas=4 deployment/test-autoscaler
```

Output:

```
deployment.apps/test-autoscaler scaled
```

Now we can check the running pods again:

```shell
kubectl get pod
```

Output:
```
NAME                                  READY   STATUS    RESTARTS   AGE
test-autoscaler-7db49f8dc5-9qmf2      1/1     Running   0          16s
test-autoscaler-7db49f8dc5-dm6fj      1/1     Running   0          3m13s
test-autoscaler-7db49f8dc5-ghpdr      1/1     Running   0          21h
test-autoscaler-7db49f8dc5-rz4l5      1/1     Running   0          16s
```

And we can check the nodes:

```shell
kubectl get nodes
```

Output:
```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-30-31.us-west-1.compute.internal    Ready    <none>   6d    v1.19.15-eks-9c63c4
ip-192-168-70-135.us-west-1.compute.internal   Ready    <none>   6d    v1.19.15-eks-9c63c4
ip-192-168-73-32.us-west-1.compute.internal    Ready    <none>   6d    v1.19.15-eks-9c63c4
```

We can check the deployment logs:

```shell
kubectl -n kube-system logs deployment.apps/cluster-autoscaler
```

### CloudWatch and EKS
We can enable or disable CloudWatch logs.

Logs types:
* Audit
* Authenticator
* API
* ControllerManager
* Scheduler

#### Enable logging

```shell
eksctl utils update-cluster-logging --config-file cloudwatch/logging/cluster.yaml --approve
```

Sample output:

```
2022-07-20 15:37:39 [ℹ]  eksctl version 0.51.0
2022-07-20 15:37:39 [ℹ]  using region us-west-1
2022-07-20 15:37:39 [ℹ]  will update CloudWatch logging for cluster "basic-eks-cluster" in "us-west-1" (enable types: api, audit, authenticator & disable types: controllerManager, scheduler)
2022-07-20 15:37:40 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:37:57 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:38:14 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:38:33 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:38:34 [✔]  configured CloudWatch logging for cluster "basic-eks-cluster" in "us-west-1" (enabled types: api, audit, authenticator & disabled types: controllerManager, scheduler)

```

Now we can check the log group in CloudWatch:
https://us-west-1.console.aws.amazon.com/cloudwatch/home?region=us-west-1#logsV2:log-groups

```
/aws/eks/basic-eks-cluster/cluster
```

#### Disable logging

```shell
eksctl utils update-cluster-logging --cluster=basic-eks-cluster --disable-types all --approve --region us-west-1
```

Output:
```
2022-07-20 15:45:28 [ℹ]  eksctl version 0.51.0
2022-07-20 15:45:28 [ℹ]  using region us-west-1
2022-07-20 15:45:28 [ℹ]  will update CloudWatch logging for cluster "basic-eks-cluster" in "us-west-1" (no types to enable & disable types: api, audit, authenticator, controllerManager, scheduler)
2022-07-20 15:45:30 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:45:46 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:46:03 [ℹ]  waiting for requested "LoggingUpdate" in cluster "basic-eks-cluster" to succeed
2022-07-20 15:46:03 [✔]  configured CloudWatch logging for cluster "basic-eks-cluster" in "us-west-1" (no types enabled & disabled types: api, audit, authenticator, controllerManager, scheduler)
```

### HELM Package Manager

<!-- 
What is HELM
-->

<!-- 
Chart: bundled of everything is needed to create the Kubernetes application (example, libraries)
Release: a running instance of a chart
-->

#### Install HELM

We are going to `install HELM`

```shell
brew install helm
```

#### Add a new repository

```shell
helm repo add stable https://charts.helm.sh/stable
```

Output:

```
"stable" has been added to your repositories
```

#### Install a chart

```shell
helm install redis-test stable/redis
```

The output will include the information to connect to the Redis server.

If you check the runnings pods, you are going to see something like:

```shell
kubectl get pod

NAME                               READY   STATUS    RESTARTS   AGE
redis-test-master-0                1/1     Running   0          3m17s
redis-test-slave-0                 1/1     Running   0          3m17s
redis-test-slave-1                 1/1     Running   0          2m40s
test-autoscaler-7db49f8dc5-c64gn   1/1     Running   0          4m24s
```

And we can check the `helm releases`

```shell
helm list

NAME      	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART       	APP VERSION
redis-test	default  	1       	2022-07-25 14:03:14.606691 -0700 PDT	deployed	redis-10.5.7	5.0.7 
```

### EKS and users

We are going to have 2 types of users:

1. Cluster Admin
2. Read only User for dedicated namescpace

#### Adding Admin user to our configmap

##### Get configmap 

We are going to get our configmap and save the output to a file so we can edit it:

```shell
kubectl -n kube-system get configmap aws-auth -o yaml > eks/aws-auth-confirmap.yaml
```

Now we are going to edit `aws-auth-confirmap.yaml` and add our admin user (created via CFN)

```yaml
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/eks-admin-user
      username: eks-admin-user
      groups:
        - system:master
```

##### Apply changes to configmap

```shell
kubectl apply -f eks/aws-auth-confirmap.yaml
```

Now we can check if we have our user under `aws-auth`

```shell
kubectl -n kube-system describe cm aws-auth
```

##### Edit `aws-auth-confirmap.yaml`

We should see something like:

```shell
mapUsers:
----
- userarn: arn:aws:iam::your-account-id:user/eks-admin-user
  username: eks-admin-user
  groups:
    - system:master
```

Previously, we configured our AWS EKS admin user: `eks-admin-user`
Check which profile you are using:

```shel
aws sts get-caller-identity
```

To switch to another profile (in this case, our EKS admin user)

```shell
export AWS_PROFILE="eks-admin"
```

```shell
aws sts get-caller-identity   

{
    "UserId": "AIDAYQ5AHLCL7OKTUSLDB",
    "Account": "your-aws-account-id",
    "Arn": "arn:aws:iam::your-aws-account-id:user/eks-admin-user"
}

```

We can check that everything works as expected:

```shell
kubectl get nodes       


NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-24-167.us-west-1.compute.internal   Ready    <none>   19h   v1.22.9-eks-810597c
ip-192-168-36-74.us-west-1.compute.internal    Ready    <none>   19h   v1.22.9-eks-810597c
```

**Run NGINX**

```shell
kubectl run nginx --image=nginx --restart=Never
```

Sample output:

```
pod/nginx created
```

We can also run a pod on a particular namespace (example: development)

```shell
kubectl run nginx --image=nginx --restart=Never -n development
```

We can check our pods:

```shell
kubectl get pods                               
NAME                               READY   STATUS    RESTARTS   AGE
nginx                              1/1     Running   0          38s
test-autoscaler-7db49f8dc5-c64gn   1/1     Running   0          19h
```

#### Adding read-only user for a dedicated namescpace

##### Create namespace

```shell
kubectl create namespace development
```

Output:

```shell
namespace/development created
```

##### Create role for the user
Check: eks/role-for-user.yaml

Note: Remember we previously created our user via CFN.

```shell
kubectl apply -f eks/role-for-user.yaml
```

Output:

```
role.rbac.authorization.k8s.io/eks-user created
```

##### Create role binding
Check: eks/role-binding.yaml

```shell
kubectl apply -f eks/role-binding.yaml
```

Output:

```
rolebinding.rbac.authorization.k8s.io/eks-user-binding created
```

##### Get configmap 

We are going to get our configmap and save the output to a file so we can edit it:

```shell
kubectl -n kube-system get configmap aws-auth -o yaml > eks/aws-auth-confirmap.yaml
```

Now we are going to edit `aws-auth-confirmap.yaml` and add our regular user (created via CFN)

```yaml
  mapUsers: |
    - userarn: arn:aws:iam::your-aws-account-id:user/eks-admin-user
      username: eks-admin-user
      groups:
        - system:masters
    - userarn: arn:aws:iam::your-aws-account-id:user/eks-user
      username: eks-user
      groups:
        - eks-user-role
```

##### Apply changes to configmap

```shell
kubectl apply -f eks/aws-auth-confirmap.yaml
```

Now we can check if we have our user under `aws-auth`

```shell
kubectl -n kube-system describe cm aws-auth
```

Output:

```
configmap/aws-auth configured
```

Switch to `eks-user` profile

```shell
export AWS_PROFILE="eks-user"
```

If we try to retrieve the pods:

```shell
kubectl get pods
```

We should see something like:

```shell
Error from server (Forbidden): pods is forbidden: User "eks-user" cannot list resource "pods" in API group "" in the namespace "default"
```

However, this user should be able to `list` pods in the `development` namespace

```shell
kubectl -n development get po
```

Output:

```
No resources found in development namespace.
```

Remember: this user is able to `LIST` but `NOT TO RUN any pod` (for example, NGINX). So, if we want to start a pod, we need to do it with our `eks-admin` user.


### Kubernetes Dashboard

#### Install Kubernetes Dashboard

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
```

Output:

```
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

#### Create Admin service account and cluster RoleBinding

```shell
kubectl apply -f eks/dashboard-admin-service-account.yaml
```

Output:

```
serviceaccount/eks-admin created
clusterrolebinding.rbac.authorization.k8s.io/eks-admin created
```

#### Create An Authentication Token (RBAC)

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

Sample output:

```
Name:         eks-admin-token-9qwzh
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: eks-admin
              kubernetes.io/service-account.uid: 1668ae9a-04fc-4dfc-85c5-a73520662aba

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImN0UVRpMW4xSTZjQ0ZlRHBQMFZ5bkVjUHNEYTEzRGJwY3FPT0FWMkdsOFkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJla3MtYWRtaW4tdG9rZW4tOXF3emgiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZWtzLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMTY2OGFlOWEtMDRmYy00ZGZjLTg1YzUtYTczNTIwNjYyYWJhIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmVrcy1hZG1pbiJ9.JusdsMtQaJQU9ch7ZxKQs1bTTGqUIUcEDzkyFR12NFUOjW1m2IBwJha03fO0aXSeACp42tgvnAiiYW1quaHp7Tt4zQbtjm3gSvB6ww14FU68gWCj0gJG9tfIgCHXlf8qp0ulKVVKY7pKz3V8bFjV1jUiVbr-HCQZxyi5nMiRQWfJ8JZXJv3rmF-9Bedx2ohPqk87LkIUS2m2yXTfPb1JQ3j-9vDixPbZCAJaFPxOz813S6IVJDXAqWd1g7NEhQoB7VF5BT9bOjoH3CVN5YVOL_BZ7_UGxjU-RKuf8LF2y8MvaDcwnT2_Eu2fB46H6wkdWdY_R1KGcVUU5HgAIhKf1Q
```

#### Access the Dashboard

To access Dashboard from your local workstation you must create a secure channel to your Kubernetes cluster. Run the following command:

```shell
kubectl proxy
```

Now access Dashboard at:

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Paste the previously generated `TOKEN`

You should see something like this...

![Kubernetes Dashboard](kubernetes-dashboard.png "Kubernetes Dashboard")





---

## Clean up

### EKS

#### Helm

**Uninstall Redis**

```shell
helm uninstall redis-test
```

### Delete user password from parameter store

```shell
aws ssm delete-parameter --name EKSUserPassword --region us-west-1
```

### Delete the EC2 Key Pair

```shell
aws ec2 delete-key-pair --key-name EKSProjectEC2KeyPair --region us-west-1
```