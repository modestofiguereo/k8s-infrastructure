# Deploying a Highly Available/Production Ready Kubernetes Cluster

## 1) Installing AWS CLI

If you already have AWS CLI you can skip this section.

#### First let's get PIP
**NOTE:** Depending on your env maybe you should replace _.profile_ with _.bash_profile_, or _.bash_login_.
```
curl -O https://bootstrap.pypa.io/get-pip.py;
python get-pip.py --user;
echo "export PATH=~/.local/bin:$PATH" | tee -a ~/.profile;
source ~/.profile;
pip --version;
```

#### Now we can proceed
```
pip install awscli --upgrade --user;
aws --version;
```

## 2) Installing Kops
Now let's get the real melma [Kops](https://github.com/kubernetes/kops):
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64;
chmod +x ./kops;
sudo mv ./kops /usr/local/bin/;
```

## 3) Installing kubectl
We need [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) in order to interect with the kubernetes cluster:

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl;
chmod +x ./kubectl;
sudo mv ./kubectl /usr/local/bin/kubectl;
```

## 3) Create kops user

```
aws iam create-group --group-name kops; # Create group for kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops;     # Give permission for EC2
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops; #Give permission for Route53
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops;      #Give permission for S3
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops;           #Give permission for IAM
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops;     #Give permission for the VPC
aws iam create-user --user-name kops;                                                                       #Create user for kops
aws iam add-user-to-group --user-name kops --group-name kops;                                               #Add user kops to the group kops
aws iam create-access-key --user-name kops;                                                                 #Generate access key
```

You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

## 4) Create K8s cluster

Put the name of your cluster and the state store in these variables:
```
export NAME=<YOUR_CLUSTER_NAME>
export KOPS_STATE_STORE=<YOUR_S3_STORE_BUCKET_NAME>
```

Create the state store bucket:
```
aws s3api create-bucket \
    --bucket prefix-example-com-state-store \
    --region us-east-1
    # --create-bucket-configuration LocationConstraint=<region> # for regions other than us-east-1.

# kops recomends using versioning
aws s3api put-bucket-versioning --bucket prefix-example-com-state-store  --versioning-configuration Status=Enabled
```

Create a HA cluster (you can find more information on these flags on kops documenetation):
```
kops create cluster \
     --master-count=3 \
     --master-size t2.small \
     --node-count 5 \
     --node-size t2.micro \
     --zones us-east-2a,us-east-2b,us-east-2c \
     --master-zones us-east-2a,us-east-2b,us-east-2c \
     --cloud-labels "Client=GBH,PartOf=xhacluster,CreatedBy=Modesto Figuereo" \
     ${NAME}
```

#### Add policy types to the cluster

Execute: `kops edit cluster --name ${NAME}``

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
             "route53:FullAccess",
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

Apply the changes with `kops update cluster --name ${NAME} --yes`.

**NOTE: you can tweak the details (for example, min and max nodes for auto scaling) of you instance group by executing `kops edit ig <instance group name>`**

## 5) Give access to your team members (optional)

Copy the key and certificates from the kops configuration:
```
export KEY=6640472233551988726606897286.key;
export CRT=6640472233551988726606897286.crt;
export MASTER_CRT=6640472252869426228642146829.crt;

aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/private/ca/${KEY} ca.keY;
aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/issued/ca/${CRT} ca.crt;
aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/issued/master/${MASTER_CRT} ca_master.crt
```

Generate certificates for the user:
```
openssl genrsa -out user.key 4096
openssl req -new -key user.key -out user.csr -subj '/CN=<USERNAME>/O=devops'
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out user.crt -days 365
```

Bind the role for that user in your cluster (here I'm binding admin role but you can bind a role that restrict user capabilities):
```
kubectl create clusterrolebinding user-cluster-admin-binding --clusterrole=cluster-admin --user=angel -n kube-system
```

### Create a kubeconfig for the user

**NOTE: After generating the config file send it through a secure channel the user.**
```
kubectl config set-cluster xhacluster.gbhplayground.com \
--server=https://api.xhacluster.gbhplayground.com \
--certificate-authority=./ca_master.crt \
--embed-certs=true \
--kubeconfig=~/path/to/config

kubectl config set-credentials xhacluster.gbhplayground.com \
--client-certificate=./user.crt \
--client-key=./user.key \
--embed-certs=true \
--kubeconfig=~/path/to/config

kubectl config set-context xhacluster.gbhplayground.com \
--cluster=xhacluster.gbhplayground.com \
--user=xhacluster.gbhplayground.com \
--kubeconfig=~/path/to/config
```

## 6) Deploy jenkins (@TODO: ADD THE TUTORIAL HERE)
```
kubectl -n kube-system create sa jenkins
kubectl create clusterrolebinding jenkins --clusterrole cluster-admin --serviceaccount=jenkins:default
```

- @TODO: Add section for kube-ingress-aws-controller deployment
- @TODO: Add section for skipper deployment
- @TODO: Add section for external-dns deployment
