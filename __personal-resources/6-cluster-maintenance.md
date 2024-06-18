# [Cluster Maintenance (Section 6)](Kubernetes-CKA-0500-Cluster-Maintenance-v1.2.pdf)

## 120. cluster maintenance - section introduction

- topics 
  - upgrade OS, upgrade cluster
  - k8s versions and releases
  - backup, restore
  - DR

## 121. download presentation deck

## 122. os upgrades

- if pod on a node is down for > 5m, it is terminated from the node
  - `kube-controller-manager --pod-eviction-timeout=5m0s ...`
  - if part of a RS, it will be recreated on another node
  - now imagine you have a blue pod and a green pod on one node
    - imagine the blue pod is part of a RS, but the green pod is not
      - what happens when the green pod dies?
        - after 5m, the blue pod gets put on another node, but the green pod does not
- `k drain <node-name>` removes all pods to other nodes (graceful termination + recreation)
  - the node is also `cordon`ed = marked unschedulable, so  
    - now you can do some maint on the drained node
      - node comes back up after maint, `k uncordon <none-name>` to allow pods back on it
- `k cordon <node-name>` simply marks node unschedulable, does not drain it

## 123. practice test - os upgrades

## 124. solution - os upgrades

- `drain` cannot delete pods that are not managed by: ReplicationController, RS, Job, DS, or StatefulSet

## 125. kubernetes software versions

- v1.11.3
  - "v1" = major version
  - "11" = minor version - released every few months, new functionality
  - "3"  = patch version - released often, usually bug fixes
- alpha --> beta --> stable release

## 126. references

- links (not needed for CKA exam):
  - [https://kubernetes.io/docs/concepts/overview/kubernetes-api/](The Kubernetes API)
  - [https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md](API Conventions)
  - [https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md](Changing the API)

## 127. cluster upgrade process

- let's focus on the core components:
  - `kube-apiserver` - nothing should have higher version than this
  - `kube-controller-manager` - can be -1 version to `kube-apiserver`
  - `kube-scheduler` - can be -1 version to `kube-apiserver`
  - `kubelet` - can be -2 versions to `kube-apiserver`
  - `kube-proxy` - can be -2 versions to `kube-apiserver`
  - `kubectl` - this is the only thing which _can_ be higher V than `kube-apiserver`

☝️ do not need to all be same versions. the version skew allows us to carry out live upgrades

![versions-1](versions-1.png)

- so when should you upgrade?
  - ex: you are on K8s v1.10
    - k8s always supports only the most recent 3 minor versions ... so ... once v1.13 comes out, you are no longer using a supported version.
    - best practice is to upgrade stepwise, one minor v at a time, and do it before 1.13 comes out so you are in support during the upgrade:
      - v1.10 --> v1.11 --> v1.12 now when v1.13 comes out, you are always under support
  - if you are on a managed service, like gcp, probably just some clicks to upgrade k8s
  - if you built it yourself with kubeadm, then 
    - `kubeadm upgrade plan`
    - `kubeadm upgrade apply`
  - if you built it "the hard way" and everything is a service, you have to manually upgrade it all
- upgrading cluster is done in 2 major steps:
  - 1. upgrade the master node(s)
    - workloads continue to serve requests, but no management functions run while compenents are down:
      - can't add stuff, can't delete stuff, can't use kubectl, etc.
      - controller mgr !work, so if a pod fails, it won't be recreated
  - 2. upgrade the worker node(s)
    - strats to upgrade worker nodes:
      - [a] upgrade all at once (breaks your app)
      - [b] rolling:
        - upgrade node01, transfers all pods to other nodes,
        - upgrade node02, transfers all pods to other nodes,
        - etc.
      - [c] add new nodes to the cluster which have the updates already
        - move workloads to new nodes and then remove old nodes
- `kubeadm plan` will tell you versions of everything and what you need to do, and give you commands to run
  - ex: upgrade recipe from v1.11 to v1.12
    - `apt-get upgrade -y kubeadm=1.12.0-00`
    - `kubeadm upgrade apply v1.12.0`
    - now `k get nodes` will still show old versions because kubelet hasn't been upgraded yet ... and it's not showing the version of the api server itself
    - now upgrade kubelets: 
      - on the master node: 
        - `apt-get upgrade -y kubeadm=1.12.0-00`
        - `kubeadm upgrade apply v1.12.0`
        - `apt-get upgrade -y kubelet=1.12.0-00`
        - restart kubelet service: `systemctl restart kubelet`
        - now `k get nodes` shows the node is upgraded
      - on the worker nodes:
        - `k drain <node-name>` to move the pods to other nodes so you can work on this one
        - `apt-get upgrade -y kubeadm=1.12.0-00`
        - `apt-get upgrade -y kubelet=1.12.0-00`
        - `k upgrade node config --kubelet-version v1.12.0`
        - restart kubelet service: `systemctl restart kubelet`
        - now mark it as schedulable again: `k uncordon <node-name>`
        - now we drain the next node, and the pods from there will find there way to our updated node, node01, etc.
  - so it's the same for master and workers, except with workers you need to be sure to move workloads first ... 
  - the goal is ofc to upgrade the cluster without taking down your apps.

## 128. demo - cluster upgrade

- follows [this](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) from the official docu
- read the notes too ... for example, about anything breaking which may have changed in between versions
- `cat /etc/*release*` <-- what distro am i using?
  - useful for example for things like [when the pkg repo changed](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#changing-the-package-repository), in which case there are diffs between OSs in how you'd accommodate that change.
- `sudo apt-cache madison kubeadm` <-- will list the available versions ... probably just pick the highest one to go to
- we update kubeadm first because we will use kubeadm to update the cluster itself ...
- when you do `k get node`, the VERSION shown is the _kubelet_ version
- big thing is just to follow the directions on the website ... walks you through the proper procedure.

## 129. practice test - cluster upgrade

## 130. solution - cluster upgrade

## 131. backup and restore methods

- _what_ should we back up? (are these options exclusive? i.e., we either backup resource cfgs, OR we backup etcd?)
  - well, what have we created?
    - resource cfgs (pod, svc, etc. defn files)
      - but we may have created these imperatively or declaratively
        - if we did it imperatively, now we have to export those to yamls, OR do this to get everything,
          whether it was created imperatively or declaratively:
            - `k get all --all-namespaces -o yaml > all-deploy-services.yaml`
        - so best practice is ofc to do everything declaratively, and store in GitHub
          - if you have this, and you lose your cluster, you can at least redeploy your entire app, just from these files
    - etcd has all cluster-related info, like state
      - so we need to back up etcd server itself ... which we can do by cfg-ing a backup tool to backup 
        the etcd configuration files dir (ex: `--data-dir=/var/lib/etcd`)
      - etcd also has a built in snapshot: `ETCDCTL_API=3 etcdctl snapshot save snapshot.db`
      - `ETCDCTL_API=3 etcdctl snapshot status snapshot.db`
      - now, to restore an etcd from this snapshot:
        - stop kube-apiserver, because cycling it will require etcd to cycle: `service kube-apiserver stop`
        - restore: `ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd-from-backup`
          - etcd will start everything over so nobody can accidentally join the existing cluster
          - now, re-cfg the etcd configuration files dir (ex: `--data-dir=/var/lib/etcd-from-backup`)
          - now, restart things:
            - `systemctl daemon-reload`
            - `service etcd restart`
            - `service kube-apiserver start`
      - ☝️ for all those etcd commands, if TLS is enabled, it's mandatory to also include the following flags:
        - `--endpoints=https://127.0.0.1:2379`
        - `--cacert=/etc/etcd/ca.crt`
        - `--cert=/etc/etcd/etcd-server.crt`
        - `--key=/etc/etcd/etcd-server.key`
    - do we have persistent volumes?
- you can alternatively use things like velero to do backups thru the k8s api
- if you are using a CSP to manage this, you likely won't anyway have access to etcd, so you'll have to back up via
  resource cfgs (yaml files).

## 132. working with etcdctl

## 133. practice test - backup and restore methods

- /etc/kubernetes/pki/etcd/server.crt
- /etc/kubernetes/pki/etcd/ca.crt
- ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db

## 134. solution - backup and restore

- `k describe pod <etcd-pod-name> | grep -i image` is another way to get etcd version
- `k describe pod <etcd-pod-name> | grep -i listen-client` to see what addy to reach etcd cluster from controlplane node
- for the cert files, etc: `k describe pod <etcd-pod-name> | grep -i cert-file` 
- remember from the name, it seems to be a static pod, and static pod defns are located in `/etc/kubernetes/manifests`
  in this case `/etc/kubernetes/manifests/etcd.yaml`
  - in [.spec.volumes.name] you can see "etcd-certs", so it's pointing out where the certs are for etcd
    - in [.spec.containers.volumeMounts.mountPath] you can see the same ...
      - this is telling us that the files from the controlplane node will be mounted to the path on the ctrs.
  - same thing with "etcd-data"
- don't forget to set the ETCDCTL_API=3 during the test!
  - if you don't, you won't even see help menu
- full command:
```
ETCDCTL_API=3 etcdctl snapshot save 
                                       --endpoints=127.0.0.1:2379 
                                       --cacert=/etc/kubernetes/pki/etcd/ca.crt
                                       --cert=/etc/kubernetes/pki/etcd/server.crt
                                       --key=/etc/kubernetes/pki/etcd/server.key
                                       /opt/snapshot-pre-boot.db
```
- now restore it. whereas when we took the backup, we needed to contact the server, so we needed all that info ^^,
  here's we're just restoring this file locally - not comms with etcd server:
```
ETCDCTL_API=3 etcdctl snapshot restore 
                                       --data-dir /var/lib/<some-new-dir>
                                       /opt/snapshot-pre-boot.db
```
- now edit the etcd cfg to point to where the backed up data are:
  - `vi /etc/kubernetes/manifests/etcd.yaml` change [.spec.volumes.hostPath.path] to where the backup is
    - now etcd will restart because this file was changed, and it'll look in where our restored backup file is 
      instead of where it was backed up from, as its data. so we effectively replaced the data in etcd.
- check that it is back up and running with a few kubectl commands ... if no response, it is not running
  - in that case, delete the etcd pod again and it'll come back, then try again with the kubectl commands

## 135. practice test backup and restore methods 2

- which clusters do i have access to?
  - `vi ~/.kube/config` look for "-cluster", OR
  - `k config get-clusters`, OR
  - `k config view`
- how many nodes are on just _one_ of those clusters?
  - switch context and check: 
    - `k config use-context cluster1`
    - `k get nodes --no-headers | wc -l`
- what is "stacked"/"external" etcd? mult. questions on this, but was never referenced in video ...
- `ssh etcd-server` 

## 136. solution: backup and restore 2

- "stacked" seems to mean it is running off the control plane node itself ...
  - another way to tell is if we do a describe on the apiserver and it points to localhost, then it is stacked etcd:
    - `k describe pod <kube-apiserver-pod-name> -n kube-system | grep etcd-servers`
- "external" means it is not running on the control plane ... you could check for the pod, not there,
  you could check for the cfg file in /manifests dir, not there ... only way to tell now is 
  that k describe kube-apiserver as above and check the ip addy of the etcd server
- can also get info about etcd server thru the process: `ps -ef | grep -i etcd` ... will show data dir, etc.
- to figure out how many nodes are part of the etcd cluster:
```
ETCDCTL_API=3 etcdctl 
                      --endpoints=... 
                      --cacert=/path/to/ca.crt
                      --cert=/path/to/server.crt
                      --key=/path/to/server.key
                      member list

# all those vars / options come from grepping the etcd process.
```
- make sure backups are owned by `etcd:etcd`
- `vi /etcd/systemd/system/etcd.service`
- `systemctl daemon-reload`
- `systemctl restart etcd`
- `systemctl status etcd`
- last thing is to restart any other control plane components to ensure they are not using stale data:
  - `k delete pods <controller-manager-name> ... <scheduler>`
- now also on control plane node `systemctl restart kubelet`

## 137. certification exam tip!
 
- on the exam, it will not give continuous, immediate feedback as in this course. 
  start checking everything yourself so you get in the habit of knowing it is right before moving on.

## 138. references

- [Backing up an etcd cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [etcd recovery](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md)
- [Disaster Recovery for your Kubernetes Clusters](https://www.youtube.com/watch?v=qRPNuT080Hk) <-- made by velero 
