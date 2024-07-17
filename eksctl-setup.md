### Prerequisites

    AWS CLI: Ensure that the AWS CLI is installed and configured with your credentials.
    eksctl: Ensure that eksctl is installed. You can download and install it from the eksctl GitHub releases page.


### Step 1: Install the Required Tools

    1.  Install AWS CLI:
```
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

    2.  Configure the AWS CLI:
```
aws configure
```

    3.  Install eksctl:

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
    4.  Verify Installation:
```
    eksctl version
```

###  Step 2: Create an IAM Role for EKS Node Group

    1.  Create IAM Role:
    Create an IAM role that will be attached to the EKS managed node group.
```
aws iam create-role --role-name EKSNodeGroupRole --assume-role-policy-document file://trust-policy.json
```

Example trust-policy.json:

```

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

    2.  Attach Policies:
Attach the necessary policies to the role.
```
aws iam attach-role-policy --role-name EKSNodeGroupRole --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name EKSNodeGroupRole --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name EKSNodeGroupRole --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```
    3.  Get the IAM Role ARN:

```
    aws iam get-role --role-name EKSNodeGroupRole --query 'Role.Arn' --output text
```

###  Step 3: Create VPC and Security Groups

You can create a VPC and security groups using a Terraform script (vpc.tf) or manually through the AWS console. Ensure the VPC and security groups are set up properly.

###  Step 4: Create the EKS Cluster Configuration File

Create a cluster configuration file (cluster-config.yaml). This file will include the IAM role, VPC configuration, and security groups.

```yaml

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: us-west-2

vpc:
  id: "vpc-xxxxxxxx"
  subnets:
    public:
      us-west-2a:
        id: "subnet-xxxxxxxx"
      us-west-2b:
        id: "subnet-yyyyyyyy"
    private:
      us-west-2a:
        id: "subnet-zzzzzzzz"
      us-west-2b:
        id: "subnet-wwwwwwww"

nodeGroups:
  - name: managed-node-group
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
    iam:
      instanceRoleARN: arn:aws:iam::123456789012:role/EKSNodeGroupRole
    ssh:
      allow: true
      publicKeyName: my-key-pair
    securityGroups:
      withShared: true
      withLocal: true
      attachIDs:
        - sg-xxxxxxxx
    tags:
      environment: production
```

###  Step 5: Create the EKS Cluster

Run the following command to create the cluster with the specified configuration:

```
eksctl create cluster -f cluster-config.yaml
```

This command will create the EKS control plane, VPC, managed node group, and other resources as specified in the configuration file. The process may take several minutes.

###  Step 6: Verify the Cluster and Node Group

After the cluster is created, verify it using the following commands:

    1.  List Clusters: eksctl get cluster
    2.  List Node Groups:   eksctl get nodegroup --cluster=my-eks-cluster
    3.  Check Nodes:    kubectl get nodes

### Step 7: Generate and Configure kubeconfig

When you create the cluster with eksctl, it automatically updates your kubeconfig file. If you need to update it manually, use the following command:
```
aws eks --region us-west-2 update-kubeconfig --name my-eks-cluster
```

This command updates your kubeconfig file with the new cluster information, allowing you to use kubectl to manage your cluster.