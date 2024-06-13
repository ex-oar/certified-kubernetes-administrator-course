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

- used to group and select objs
- @pod spec: [.metadata.labels] then `k get pods --selector key=val`
- @RS  spec: [.metadata.labels] === [.spec.template.metadata.labels] because 
  the [.spec.template] is ofc a pod spec for pods in that RS (?)
```
The ReplicaSet "replicaset-1" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"tier":"nginx"}: `selector` does not match template `labels`
```
  - [.metadata.labels] are there for _other_ objs to discover the RS
  - [.spec.selector.matchLabels] is for the RS to know what pods it controls
    by matching these labels on pods. so:
    RS[.spec.selector.matchLabels] === Pod[.metadata.labels]
- same for Service
- annotations are for other generic metadata [.metadata.annotations]

## 57. practice test - labels and selectors

- `kubectl get pods --selector=env=dev --no-headers | wc -l`
- `kubectl get pods --selector=env=prod,bu=finance,tier=frontend`

## 58. solution: labels and selectors

## 59. taints and tolerations

- restrict what pods go on what nodes
- imagine a person puts on bug spray - this person is a _node_ and the spray is a _taint_ 
  - the bug (a _pod_) is intolerant to the smell, so does not land on the person (_node_)
  - but - there could be other bugs that aren't affected by this particular spray,
    so they land on the person anyway. these bugs (_pod_s) _tolerate_ (are unaffected by) the _taint_ (bug spray)
  - so two things detm if a pod can be placed on a node: 
    - the node's taint
    - the pod's toleration to that particular node's taint
- example
  - by default, we'll try to equally spread pods across nodes
  - now assume dedicated resources on node 1
    - this could be for a particular use case, app, etc.
    - so we want only pods that belong to that particular use case (app) to be placed on node 1
      - first, place a taint on node 1 to prevent _all pods_ from being scheduled on that node
      - by default, pods don't have tolerations, so by default, no pods can tolerate that taint
      - so now we place a toleration on particular pods ... now scheduler will place only those
        particular pods with tolerations that match the taint on the node on the node
- `kubectl taint nodes <node-name> key=val:<taint-effect>`
  - <taint-effect> is what happens with pods that don't tolerate the taint. 3 effects:
    - `NoSchedule`
    - `PreferNoSchedule` - best effort don't place this pod on this node
    - `NoExecute` - default - don't put new pods without tolerations on this node, and existing pods without toleration evicted. this could happen, for example, if a taint was applied to a node after it was already humming along with pods on it.
    - example:
      - `kubectl taint nodes node1 app=blue:NoSchedule`
- to add the matching toleration to a pod:
```
...
spec:
  ...
  tolerations:
  - key: "app"
    value: "blue"
    operator: "Equal"
    effect: "NoSchedule"
...
```
- taints and tolerations however, do _not_ tell a pod to go to a particular node.
  a pod with a toleration for node1 may end up going to node2. all the taint and
  toleration does is say the pod _may_ go to node1. it's not guaranteed.
- to do that ^^^^, you need _node affinity_.
- control plane node doesn't get pods scheduled because k8s auto taints it on
  creation to prevent putting pods there. you could change this, but it is not good practice.
  - `k describe node kubemaster | grep Taint`
    - gives: "Taints: node-role.kubernetes.io/master:NoSchedule"

## 60. practice test - taints and tolerations

