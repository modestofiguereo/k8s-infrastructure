# Create k8s cluster

## Create kops user

```
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)

aws iam create-group --group-name kops                                                                     # Create group for kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops     # Give permission for EC2
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops # Give permission for Route53
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops      # Give permission for S3
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops           # Give permission for IAM
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops     # Give permission for the VPC

aws iam create-user --user-name kops                                                                       # Create user for kops
aws iam add-user-to-group --user-name kops --group-name kops                                               # Add user kops to the group kops

aws iam create-access-key --user-name kops                                                                 # Generate access key
```

You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now

echo "export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)" | tee -a ~/.profile
echo "export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)" | tee -a ~/.profile

source ~/.profile
```

## Create the cluster

#### Create S3 state store bucket

Put the name of your cluster and the state store in these variables:
```
export NAME=<YOUR_CLUSTER_NAME>
export KOPS_STATE_STORE_NAME=prefix-example-com-state-store
export KOPS_STATE_STORE=s3://${KOPS_STATE_STORE_NAME}
```

Create the state store bucket:
```
aws s3api create-bucket \
    --bucket ${KOPS_STATE_STORE_NAME} \
    --region us-east-1
    # --create-bucket-configuration LocationConstraint=<region> # for regions other than us-east-1.

# kops recomends using versioning
aws s3api put-bucket-versioning --bucket ${KOPS_STATE_STORE_NAME}  --versioning-configuration Status=Enabled
```

#### Create cluster

Create a HA cluster (you can find more information on these flags on kops documenetation):
```
kops create cluster \
   --master-count=3 \
   --master-size t2.small \
   --node-count 5 \                                                       
   --node-size t2.micro \
   --zones us-east-2a,us-east-2b,us-east-2c \
   --master-zones us-east-2a,us-east-2b,us-east-2c \
   --cloud-labels "kubernetes.io/cluster/${NAME}: owned" \
   ${NAME}
```

**NOTE: kops automatically creates a kubeconfig file (``~/.kube/config`) for you so you can interact with your cluster**

#### Add policy types to the cluster

Execute: `kops edit cluster --name ${NAME}`

Add this under the `spec` section:
```
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": [
             "acm:ListCertificates",
             "acm:GetCertificate",
             "acm:DescribeCertificate",
             "autoscaling:DescribeAutoScalingGroups",
             "autoscaling:DescribeLoadBalancerTargetGroups",
             "autoscaling:AttachLoadBalancers",
             "autoscaling:DetachLoadBalancers",
             "autoscaling:DetachLoadBalancerTargetGroups",
             "autoscaling:AttachLoadBalancerTargetGroups",
             "cloudformation:*",
             "elasticloadbalancing:*",
             "elasticloadbalancingv2:*",
             "ec2:DescribeInstances",
             "ec2:DescribeSubnets",
             "ec2:DescribeSecurityGroups",
             "ec2:DescribeRouteTables",
             "ec2:DescribeVpcs",
             "ec2:DescribeAccountAttributes",
             "ec2:DescribeInternetGateways",
             "iam:GetServerCertificate",
             "iam:ListServerCertificates",
             "iam:CreateServiceLinkedRole",
             "route53:GetHostedZone",
             "route53:ListHostedZonesByName",
             "route53:CreateHostedZone",
             "route53:DeleteHostedZone",
             "route53:ChangeResourceRecordSets",
             "route53:CreateHealthCheck",
             "route53:GetHealthCheck",
             "route53:DeleteHealthCheck",
             "route53:UpdateHealthCheck",
             "route53:ListHostedZones",
             "route53:ListResourceRecordSets",
             "route53:ChangeResourceRecordSets",
             "ec2:DescribeVpcs",
             "ec2:DescribeRegions",
             "servicediscovery:*"
          ],
          "Resource": ["*"]
        }
      ]
```

Apply the changes with `kops update cluster --name ${NAME} --yes`. After that the AWS is going to take sometime to set up the cluster. We can use the command `kops validate cluster` to know when the cluster is ready.

**NOTE: you can tweak the details of you instance group by executing (for example, min and max nodes for auto scaling) `kops edit ig <instance group name>`**
