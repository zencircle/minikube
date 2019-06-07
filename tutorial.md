### Install kubectl
`$brew install kubernetes-cli`

### Install minikube 
Start minikube    
`$brew cask install minikube`

### Documentation
https://kubernetes.io/docs/tutorials/hello-minikube/   
https://github.com/kubernetes/minikube/   
https://kubernetes.io/docs/setup/minikube/#quickstart  

### Start minikube
```
$minikube start (if virtualbox or simiar is  installed)
Default 2vCPU, 2GB Memory and 20G disk

😄  minikube v1.1.0 on darwin (amd64)
💿  Downloading Minikube ISO ...
 131.28 MB / 131.28 MB [============================================] 100.00% 0s
🔥  Creating virtualbox VM (CPUs=2, Memory=2048MB, Disk=20000MB) ...
🐳  Configuring environment for Kubernetes v1.14.2 on Docker 18.09.6
💾  Downloading kubelet v1.14.2
💾  Downloading kubeadm v1.14.2
🚜  Pulling images ...
🚀  Launching Kubernetes ... 
⌛  Verifying: apiserver proxy etcd scheduler controller dns
🏄  Done! kubectl is now configured to use "minikube"
```
### Working with minikube 
```
$minikube status
host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100
```
### Dashboard
`$minikube dashbard`    
http://127.0.0.1:52879/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/#!/about?namespace=default   

### Create deployment, replicaset and pods
```
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-minikube created
```
### Get Pod
```
kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
hello-minikube-78c9fc5f89-kvrxr   1/1     Running   0          7m56s
```
### Get ReplicaSet
```
kubectl get replicaset
NAME                        DESIRED   CURRENT   READY   AGE
hello-minikube-78c9fc5f89   1         1         1       8m33s
```
### Get Deployment 
```
kubectl get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           8m50s
```
### No service(not way to access the pod)

### Create service(ClusterIP and NodePort)
```
kubectl expose deployment hello-minikube --type=NodePort --port=8080
service/hello-minikube exposed
``` 
### Get services
```
kubectl get services
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-minikube   NodePort    10.101.30.146   <none>        8080:31228/TCP   5m51s
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          14m
```
### Testing Service using kubectl port-forward (Cluster-IP)
```
kubectl port-forward svc/hello-minikube 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```
http://localhost:8080   

### Test Service using "NodePort"
```
minikube service hello-minikube
```
http://192.168.99.100:31228/
### Scaling Deployment (Change replicas from 1 to 2)
```
kubectl scale --replicas=2 deployment hello-minikube
deployment.extensions/hello-minikube scaled

kubectl get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   2/2     2            2           10h
```
### Self Healing (destroy 1 of the 2 pods)
```
kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
hello-minikube-5bc47cf7b5-hpkxg   1/1     Running   1          10h
hello-minikube-5bc47cf7b5-q7mt8   1/1     Running   0          2m4s

kubectl delete pod hello-minikube-5bc47cf7b5-hpkxg
pod "hello-minikube-5bc47cf7b5-hpkxg" deleted

kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
hello-minikube-5bc47cf7b5-6584v   1/1     Running   0          21s
hello-minikube-5bc47cf7b5-q7mt8   1/1     Running   0          2m54s
```
## System Components (addons)
```
kubectl get deployment --namespace kube-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
coredns                2/2     2            2           17h
kubernetes-dashboard   1/1     1            1           17h
```
## UNDER THE HOOD
### SSH into minikube 
`ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip)`

## Control Plane
### etcd
```
$ps aux | grep etcd | grep -v kube-apiserver
root      3211  1.6  2.7 10548180 56324 ?      Ssl  15:09   0:38 etcd --advertise-client-urls=https://192.168.99.100:2379 --cert-file=/var/lib/minikube/certs/etcd/server.crt --client-cert-auth=true --data-dir=/data/minikube --initial-advertise-peer-urls=https://192.168.99.100:2380 --initial-cluster=minikube=https://192.168.99.100:2380 --key-file=/var/lib/minikube/certs/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://192.168.99.100:2379 --listen-peer-urls=https://192.168.99.100:2380 --name=minikube --peer-cert-file=/var/lib/minikube/certs/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/var/lib/minikube/certs/etcd/peer.key --peer-trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt

$docker ps  | grep -v pause | grep etcd
d01a2d64e5f2        2c4adeb21b4f           "etcd --advertise-cl…"   30 minutes ago      Up 30 minutes                           k8s_etcd_etcd-minikube_kube-system_18c827a17f0a6b507c2029890cd786ad_2
```
###  apiserver
```
$ ps aux | grep kube-apiserver
root      3205  3.1 12.3 406084 251292 ?       Ssl  15:09   1:11 kube-apiserver --advertise-address=192.168.99.100 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --enable-bootstrap-token-auth=true --etcd-cafile=/var/lib/minikube/certs/etcd/ca.crt --etcd-certfile=/var/lib/minikube/certs/apiserver-etcd-client.crt --etcd-keyfile=/var/lib/minikube/certs/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/var/lib/minikube/certs/apiserver-kubelet-client.crt --kubelet-client-key=/var/lib/minikube/certs/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/var/lib/minikube/certs/front-proxy-client.crt --proxy-client-key-file=/var/lib/minikube/certs/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=8443 --service-account-key-file=/var/lib/minikube/certs/sa.pub --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/var/lib/minikube/certs/apiserver.crt --tls-private-key-file=/var/lib/minikube/certs/apiserver.key

$docker ps  | grep -v pause | grep apiserver
b308f5220ac4        5eeff402b659           "kube-apiserver --ad…"   31 minutes ago      Up 31 minutes                           k8s_kube-apiserver_kube-apiserver-minikube_kube-system_59681f267057a4cf6f6fd79d17776414_2
```
### kube-controller-manager
```
$docker ps  | grep -v pause | grep control
e87f067aa60f        8be94bdae139           "kube-controller-man…"   35 minutes ago      Up 35 minutes                           k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_9c1e365bd18b5d3fc6a5d0ff10c2b125_2
```
### kube-scheduler
```
$ps aux | grep kube-scheduler
root      3395  0.1  1.7 141508 35676 ?        Ssl  15:09   0:04 kube-scheduler --bind-address=127.0.0.1 --kubeconfig=/etc/kubernetes/scheduler.conf --leader-elect=true
```
## Node Components
### kubelet
```
$ ps aux | grep kubelet | grep -v kube-apiserver 
root      2855  3.8  4.7 1361508 97300 ?       Ssl  15:09   1:17 /usr/bin/kubelet --allow-privileged=true --authorization-mode=Webhook --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --cgroup-driver=cgroupfs --client-ca-file=/var/lib/minikube/certs/ca.crt --cluster-dns=10.96.0.10 --cluster-domain=cluster.local --container-runtime=docker --fail-swap-on=false --hostname-override=minikube --kubeconfig=/etc/kubernetes/kubelet.conf --pod-manifest-path=/etc/kubernetes/manifests
```
### kube-proxy
```
ps aux | grep kube-proxy
root      4060  0.0  1.5 138988 31984 ?        Ssl  15:09   0:02 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=minikube
```
