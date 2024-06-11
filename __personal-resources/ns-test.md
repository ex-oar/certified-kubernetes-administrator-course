```
controlplane ~ ➜  kubectl run nginx-pod --image=nginx:alpine
pod/nginx-pod created

controlplane ~ ➜  kubectl run redis --image=redis:alpine --labels=tier=db
pod/redis created

controlplane ~ ✖ kubectl create service clusterip redis-service --tcp=6379:6379
service/redis-service created

controlplane ~ ➜  kubectl create deployment webapp --image=kodekloud/webapp-color --replicas=3
deployment.apps/webapp created

controlplane ~ ➜  kubectl run custom-nginx --image=nginx --port=8080
pod/custom-nginx created

controlplane ~ ➜  kubectl create ns dev-ns
namespace/dev-ns created

controlplane ~ ➜  kubectl create deployment redis-deploy --image=redis --replicas=3 -n dev-ns
deployment.apps/redis-deploy created
```
