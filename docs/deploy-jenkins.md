# Deploy Jenkins

To deploy jenkins we need to build the image from the `jenkins/docker/jenkins.Dockerfile`, then push it to docker hub or our private registry.

After that we can perform the deployment:

```
IMAGE=<IMAGE_NAME>      # Assign the image name to this variable. (If you choosed to use a private repo, remember to prepend the name of the registry before the image name, i.e: registry/image-name:tag)
DOMAIN=<DOMAIN>         # Assign your domain.

sed -i "s/<IMAGE_NAME>/$IMAGE/" ./jenkins/jenkins-deployment.yaml
sed -i "s/<DOMAIN>/$DOMAIN/" ./jenkins/jenkins-ingress.yaml

kubectl apply -f ./jenkins/jenkins-namespace.yaml
kubectl apply -f ./jenkins/jenkins-service-account.yaml
kubectl apply -f ./jenkins/jenkins-persistent-volumen-claim.yaml
kubectl apply -f ./jenkins/jenkins-deployment.yaml
kubectl apply -f ./jenkins/jenkins-service.yaml
kubectl apply -f ./jenkins/jenkins-ingress.yaml
```

After jenkins is deployed we need to do some manual configuration:
- First of all enable security `Manage Jenkins -> Configure Global Security -> Enable Security`, then choose the options that better fits you and hit save.
- Add kubernetes cloud `Manage Jenkins -> Configure System -> Add Cloud`, fill the options.

@TODO: Insert here an screenshot as example on how to fill those options.
