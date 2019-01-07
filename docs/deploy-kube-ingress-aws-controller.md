# Deploy kube ingress aws controller

Kube Ingress AWS Controller together with Skipper and External-DNS make life easier when dealing routing traffic to your services through ingress resources. As soon as we create an ingress resource they create the DNS record in AWS for us and route the traffic in the ports and paths that are specified in the ingress resource.

### Service Accounts

Before deploying any of the above mentioned resources we need to create the service account the will give them permission to perform. We can do that with the following commands:
```
kubectl apply -f ./kube-ingress-aws-controller/skipper-aws-ingress-service-account.yaml
kubectl apply -f ./kube-ingress-aws-controller/external-dns/external-dns-service-account.yaml
```

### kube-ingress-aws-controller

##### Security Group
Before deploying the kube-ingress-aws-controller we need to create an special security group for it:

```
KOPS_CLUSTER_NAME=${NAME}

VPC_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=nodes.${KOPS_CLUSTER_NAME} | jq '.["SecurityGroups"][0].VpcId' -r)

aws ec2 create-security-group --description ingress.${KOPS_CLUSTER_NAME} --group-name ingress.${KOPS_CLUSTER_NAME} --vpc-id $VPC_ID
aws ec2 describe-security-groups --filter Name=vpc-id,Values=$VPC_ID  Name=group-name,Values=ingress.${KOPS_CLUSTER_NAME}

SG_ID_INGRESS=$(aws ec2 describe-security-groups --filter Name=vpc-id,Values=$VPC_ID  Name=group-name,Values=ingress.${KOPS_CLUSTER_NAME} | jq '.["SecurityGroups"][0]["GroupId"]' -r)
SG_ID_NODE=$(aws ec2 describe-security-groups --filter Name=vpc-id,Values=$VPC_ID  Name=group-name,Values=nodes.${KOPS_CLUSTER_NAME} | jq '.["SecurityGroups"][0]["GroupId"]' -r)

aws ec2 authorize-security-group-ingress --group-id $SG_ID_INGRESS --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID_INGRESS --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-egress --group-id $SG_ID_INGRESS --protocol all --port -1 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID_NODE --protocol all --port -1 --source-group $SG_ID_INGRESS
aws ec2 create-tags --resources $SG_ID_INGRESS --tags Key="kubernetes.io/cluster/${KOPS_CLUSTER_NAME}",Value="owned" Key="kubernetes:application",Value="kube-ingress-aws-controller"
```

##### AWS Certificate Manager (ACM)

To have TLS termination you can use AWS managed certificates. If you are unsure if you have at least one certificate provisioned use the following command to list ACM certificates:

`aws acm list-certificates`

If you have one, you can move on to the next section.

To create an ACM certificate, you have to request a CSR with a domain name that you own in route53, for example.org. We will here request one wildcard certificate for example.org:

`aws acm request-certificate --domain-name *.example.org`

You will have to successfully do a challenge to show ownership of the given domain. In most cases you have to click on a link from an e-mail sent by certificates.amazon.com. E-Mail subject will be Certificate approval for <example.org>.

If you did the challenge successfully, you can now check the status of your certificate. Find the ARN of the new certificate:

`aws acm list-certificates`

Describe the certificate and check the Status value:

`aws acm describe-certificate --certificate-arn arn:aws:acm:<snip> | jq '.["Certificate"]["Status"]'`

If this is no "ISSUED", your certificate is not valid and you have to fix it. To resend the CSR validation e-mail, you can use:

`aws acm resend-validation-email`

##### Deploy the ingress controller

Now we can deploy kube-ingress-aws-controller with the following commands:

```
AWS_REGION=[AWS REGION TO USE]   # Assign the name of the region you want to use this variable
DOMAIN=[YOUR DOMAIN]             # Assign your domain name to this variable

sed -i "s/<AWS_REGION>/$AWS_REGION/" ./kube-ingress-aws-controller/kube-ingress-aws-controller-deployment.yaml
sed -i "s/<DOMAIN>/$DOMAIN/" ./kube-ingress-aws-controller/external-dns/external-dns-deployment.yaml
sed -i "s/<AWS_REGION>/$AWS_REGION/" ./kube-ingress-aws-controller/external-dns/external-dns-deployment.yaml

kubectl apply -f ./kube-ingress-aws-controller/kube-ingress-aws-controller-deployment.yaml
```

### Skipper

Let's deploy skipper:
```
kubectl apply -f ./kube-ingress-aws-controller/skipper/skipper-deployment.yaml
```

### External-dns

Finally Let's deploy external-dns:
```
kubectl apply -f ./kube-ingress-aws-controller/external-dns/external-dns-deployment.yaml
```
