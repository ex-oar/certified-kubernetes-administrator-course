# Scheduling (Section 3)

## 51. scheduling section introduction

## 52. download presentation deck for this section

- [Kubernetes-CKA-0200-Scheduling.pdf](Kubernetes-CKA-0200-Scheduling.pdf)
- [Networking.pdf](Networking.pdf)
- [Udemy-Kubernetes-taints-tolerations.pdf](Udemy-Kubernetes-taints-tolerations.pdf)
- [Static-Pods.pdf](Static-Pods.pdf)

## 53. manual scheduling

- each pod defn has a `nodeName` property ... scheduler starts by running over the 
  pods, looking for the ones that don't have this property set = candidates for scheduling.
- it then runs thru algo and creates a binding obj by setting the node name to the one w/highest score
- you can ofc set this manually in the manifest - but only at pod create 
  - k8s won't allow manual change if pod already exists, so, if a pod already exists and you want to 
    manually assign to a node, then you need to create a binding object:
```
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: <node-name>
```

^^ you need to send this equivalent to the pod api in json format:

`curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1",:kind":...}' http://<server>/api/v1/namespaces/default/pods/<podname>/binding`

- `k get nodes`

## 54. practice test - manual scheduling

- check if there is a scheduler present:
  - `kubectl get pods -n kube-system -l component=kube-scheduler`
  - scheduler runs as a pod in the kube-system ns
- also could describe the pod and see "Node:" is "<none>"
```
controlplane ~ ➜  kubectl get pod nginx -w
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     ContainerCreating   0          6s
nginx   1/1     Running             0          8s
```

## 55. solution - manual scheduling

- `k replace --force ...` is the same as `k delete ...` + `k apply`

## 56.  labels and selectors


