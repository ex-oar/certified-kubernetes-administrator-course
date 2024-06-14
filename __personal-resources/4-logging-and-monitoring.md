# [Logging & Monitoring (Section 4)](Kubernetes-CKA-0300-Logging-Monitoring.pdf)

## 82. logging and monitoring section introduction

## 83. download presentation deck

## 84. monitor cluster components

- key is: _what_ do you want to monitor?
  - maybe node-level metrics, like:
    - num nodes in cluster
    - how many nodes healthy 
    - perf metrics of nodes like:
      - cpu 
      - memory 
      - disk use
      - network use
  - maybe pod-level metrics, like:
    - num pods
    - perf metrics of pods like:
      - cpu consumption
      - memory consumption
- k8s does not come with a built-in full-featured monitoring solution
  - so use something like: metrics server, prom, elastic stack, datadog, dynatrace, etc.
  - `metrics server` is slimmed down version of the deprecated "Heapster".
    - can have one metrics server per k8s cluster 
    - it gets metrics from all pods and nodes, aggregates and stores in memory
      - metrics are not stored on disk! so no time series possible, for ex.
      - data comes from kubelet `cAdvisor` (a subcomponent of the kubelet)
        - cAdvisor gets metrics from pods and exposing them thru kubelet api 
- if using minikube, enable metrics server: `minikube addons enable metrics-server`
- for anything else, deploy like so:
  - 1. `git clone https://github.com/kubernetes-incubator/metrics-server.git`, then
  - 2. `cd kubernetes-metrics-server`
  - 3. `k create -f .`
    - once it's been running awhile and has collected data, check out `k top node` 
      to see cpu and mem consumption for each node (likewise `k top pod` for pod metrics)

## 85. practice test - monitoring

## 86. solution: monitor cluster components

## 87. managing application logs

- logs for docker
  - `docker rn kodekloud/event-simulator` will stream fake events to simulate webserver activity
    - the logs go to stdout ... so if you put it in background with `docker run -d ...`, you'd not see any logs.
  - `docker logs -f <ctr-id>` <-- `-f` streams log live to stdout 
- logs for k8s
  - using the same image as above:
```
apiVersion: v1
kind: Pod
metadata: 
  name: event-simulator-pod
spec;
  containers:
  - name: event-simulator
    image: kodekloud/even-simulator
  - name: image-processor
    image: some-image-processor
```
- then `k create -f ...`
- then `k logs -f event-simulator-pod`
- remember: N ctrs per pod, so you need to explicitly specify the name of the ctr:
  - then `k logs -f event-simulator-pod event-simulator` <-- get the logs from ctr event-simulator on pod
  event-simulator-pod.

## 88. practice test - monitor application logs

## 89. solution: logging

