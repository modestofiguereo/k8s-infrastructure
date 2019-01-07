# Grant access to the cluster to your teammates

### Generate the certificates

Copy the key and certificates from the kops configuration:
```
export KEY=<KEY_FILENAME>.key
export CRT=<CRT_FILENAME>.crt
export MASTER_CRT=<CRT_FILENAME>.crt
export USERNAME=<USERNAME>

aws s3 cp s3://${KOPS_STATE_STORE}/${NAME}/pki/private/ca/${KEY} ca.keY
aws s3 cp s3://${KOPS_STATE_STORE}/${NAME}/pki/issued/ca/${CRT} ca.crt
aws s3 cp s3://${KOPS_STATE_STORE}/${NAME}/pki/issued/master/${MASTER_CRT} ca_master.crt
```

Generate certificates for the user:
```
openssl genrsa -out ${USERNAME}.key 4096
openssl req -new -key ${USERNAME}.key -out ${USERNAME}.csr -subj '/CN=${USERNAME}/O=devops'
openssl x509 -req -in ${USERNAME}.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ${USERNAME}.crt -days 365
```

Bind the view role to the user:
```
kubectl create clusterrolebinding viewer-cluster-admin-binding --clusterrole=view --user=${USERNAME} -n <NAMESPACE>
```

### Create a kubeconfig for the user

**NOTE: After generating the config file send it through a secure channel the your teammates.** <br />
**NOTE: You can generate different kubeconfig files for each of your teammates with different roles and for different namespaces.**
```
kubectl config set-cluster ${NAME} \
  --server=https://api.${NAME} \
  --certificate-authority=./ca_master.crt \
  --embed-certs=true \
  --kubeconfig=~/path/to/config

kubectl config set-credentials ${NAME} \
  --client-certificate=./${USERNAME}.crt \
  --client-key=./${USERNAME}.key \
  --embed-certs=true \
  --kubeconfig=~/path/to/config

kubectl config set-context ${NAME} \
  --cluster=${NAME} \
  --user=${NAME} \
  --kubeconfig=~/path/to/config     
```
