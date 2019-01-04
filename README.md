# Deploying a Highly Available/Production Ready Kubernetes Cluster

#### Requirements
- [AWS CLI](https://aws.amazon.com/cli/)
- [Kops](https://github.com/kubernetes/kops)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [OpenSSL](https://www.openssl.org/)

#### Useful
- [kubectx & kubens](https://github.com/ahmetb/kubectx)

### Content
- [Installing AWS CLI](#installing-aws-cli)
- [Installing kops](#installing-kops)
- [Installing kubectl](#installing-kubectl)
- [Create kops user](#create-kops-user)
- [Create k8s cluster](#create-k8s-cluster)
- [Grant access to your teammates](#grant-access-to-your-teammates)

## Installing AWS CLI

If you already have AWS CLI you can skip this section.

#### First let's get PIP
**NOTE:** Depending on your env maybe you should replace `.profile` with `.bash_profile`, or `.bash_login`.
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

## Installing Kops

Kops is a tool that automates the creation and maintenance of the kubernentes cluster.
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64;
chmod +x ./kops;
sudo mv ./kops /usr/local/bin/;
```

## Installing kubectl
We need kubectl in order to interect with the kubernetes cluster:

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl;
chmod +x ./kubectl;
sudo mv ./kubectl /usr/local/bin/kubectl;
```

## Create kops user

export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```
aws iam create-group --group-name kops;                                                                     # Create group for kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops;     # Give permission for EC2
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops; # Give permission for Route53
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops;      # Give permission for S3
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops;           # Give permission for IAM
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops;     # Give permission for the VPC

aws iam create-user --user-name kops;                                                                       # Create user for kops
aws iam add-user-to-group --user-name kops --group-name kops;                                               # Add user kops to the group kops

aws iam create-access-key --user-name kops;                                                                 # Generate access key
```

You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now

echo "export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)" | tee -a ~/.profile;
echo "export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)" | tee -a ~/.profile;

source ~/.profile;
```

## Create k8s cluster

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

## Grant access to your teammates

#### Generate the certificates

Copy the key and certificates from the kops configuration:
```
export KEY=6640472233551988726606897286.key;
export CRT=6640472233551988726606897286.crt;
export MASTER_CRT=6640472252869426228642146829.crt;
export USERNAME=<USERNAME>;

aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/private/ca/${KEY} ca.keY;
aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/issued/ca/${CRT} ca.crt;
aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/issued/master/${MASTER_CRT} ca_master.crt
```

Generate certificates for the user:
```
openssl genrsa -out ${USERNAME}.key 4096
openssl req -new -key ${USERNAME}.key -out ${USERNAME}.csr -subj '/CN=${USERNAME}/O=devops'
openssl x509 -req -in ${USERNAME}.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ${USERNAME}.crt -days 365
```

Bind the view role to the user:
```
kubectl create clusterrolebinding viewer-cluster-admin-binding --clusterrole=view --user=viewer -n <NAMESPACE>
```

#### Create a kubeconfig for the user

**NOTE: After generating the config file send it through a secure channel the your teammates.** <br />
**NOTE: You can generate different kubeconfig files for each of your teammates with different roles and for different namespaces.**
```
kubectl config set-cluster xhacluster.gbhplayground.com \
--server=https://api.xhacluster.gbhplayground.com \
--certificate-authority=./ca_master.crt \
--embed-certs=true \
--kubeconfig=~/path/to/config

kubectl config set-credentials xhacluster.gbhplayground.com \
--client-certificate=./${USERNAME}.crt \
--client-key=./${USERNAME}.key \
--embed-certs=true \
--kubeconfig=~/path/to/config

kubectl config set-context xhacluster.gbhplayground.com \
--cluster=xhacluster.gbhplayground.com \
--user=xhacluster.gbhplayground.com \
--kubeconfig=~/path/to/config
```

- @TODO: Add section for kubernetes dashboard and heapster deployment
- @TODO: Add section for kube-ingress-aws-controller deployment
- @TODO: Add section for skipper deployment
- @TODO: Add section for external-dns deployment
- @TODO: Add section for Jenkins deployment
