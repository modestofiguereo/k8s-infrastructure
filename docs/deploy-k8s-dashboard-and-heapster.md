# Deploy k8s dashboard and heapster

### Kubernetes Dashboard

Kubernetes Dashboard is a handy tool for visualizing and performing basic tasks in your cluster.

To deploy Kubernetes Dashboard is as easy as:
```
kubectl apply -f ./kubernetes-dashboard/kubernetes-dashboard.yaml
```

To enter the dashboard execute: `kubectl proxy`. <br />
Now you can access the dashboard in this url: [`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`](`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`).

### Create an admin user for your dashboard

For accessing the dashboard we need an user with the right permissions, here I will create an admin user but you can create users with more restrictions if you want to give access to your teammates.

Using the admin user yaml inside the `users/` directory we can create the user and grant them admin privileges as follows:
```
kubectl apply -f ./kubernetes-dashboard/admin-user-service-account.yaml
```

Now we need the token for that user in other to login to the dashboard, we can get it with this command:
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

This will print something like this:
```
Name:         admin-user-token-6gl6l
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=b16afba9-dfec-11e7-bbb9-901b0e532516

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZnbDZsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTZhZmJhOS1kZmVjLTExZTctYmJiOS05MDFiMGU1MzI1MTYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.M70CU3lbu3PP4OjhFms8PVL5pQKj-jj4RNSLA4YmQfTXpPUuxqXjiTf094_Rzr0fgN_IVX6gC4fiNUL5ynx9KU-lkPfk0HnX8scxfJNzypL039mpGt0bbe1IXKSIRaq_9VW59Xz-yBUhycYcKPO9RM2Qa1Ax29nqNVko4vLn1_1wPqJ6XSq3GYI8anTzV8Fku4jasUwjrws6Cn6_sPEGmL54sq5R4Z5afUtv-mItTmqZZdxnkRqcJLlg2Y8WbCPogErbsaCDJoABQ7ppaqHetwfM_0yMun6ABOQbIwwl8pspJhpplKwyo700OSpvTT9zlBsu-b35lzXGBRHzv5g_RA
```

We can copy the value for token and pasted in the login screen in the Dashboard. <br />
That's it! Now you have visualization of all your resources in the cluster.

### Deploy Heapster

Heapster is a tool that allow you to visualize the cpu and memory consumption of your pods and nodes.
You can set it up by executing:

```
DOMAIN=<DOMAIN>         # Assign your domain.

sed -i "s/<DOMAIN>/$DOMAIN/" ./kubernetes-dashboard/heapster/grafana.yaml

kubectl apply -f ./kubernetes-dashboard/heapster/influxdb.yaml
kubectl apply -f ./kubernetes-dashboard/heapster/grafana.yaml
kubectl apply -f ./kubernetes-dashboard/heapster/heapster.yaml
```

After Heapster is deployed it gets something in order to get metrics to display, after that you should get some charts in your dashboard displaying cpu and memory metrics.
