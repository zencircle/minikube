```
CONFIG MAP EXAMPLE
http://pwittrock.github.io/docs/user-guide/configmap/

kubectl apply -f http://pwittrock.github.io/docs/user-guide/configmap/configmap.yaml
kubectl get configmaps test-configmap -o yaml

USE CONFIG-MAP : Create a pod that consumes a configMap in environment variables
kubectl apply -f  http://pwittrock.github.io/docs/user-guide/configmap/env-pod.yaml
kubectl get configmaps test-configmap -o yaml

SECRET EXAMPLE
http://pwittrock.github.io/docs/tasks/inject-data-application/distribute-credentials-secure/#create-a-secret

USE SECRET
kubectl apply  secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
kubectl describe secret test-secret
kubectl apply -f https://raw.githubusercontent.com/zencircle/minikube/master/secret-pod.yaml

SHELL INTO CONTAINER AND VIEW FILES
kubectl exec -it secret-test-pod -- /bin/bash
root@secret-test-pod:/# cat etc/secret-volume/username
root@secret-test-pod:/# cat etc/secret-volume/password
```
