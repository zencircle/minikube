```
VOLUMES
https://github.com/donvito/learngo/tree/master/mongo-microservice/mongodb

kubectl get storageclass
NAME                 PROVISIONER                AGE
standard (default)   k8s.io/minikube-hostpath   2d6h

kubectl describe storageclass standard | grep Provisioner
Provisioner:           k8s.io/minikube-hostpath

MONGODB PVC
kubectl apply -f https://raw.githubusercontent.com/donvito/learngo/master/mongo-microservice/mongodb/mongodata-persistentvolumeclaim.yaml
kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodata   Bound    pvc-3fb5821c-8a71-11e9-bc84-08002742a1ec   100Mi      RWO            standard       2m28s

MONGODB DEPLOYMENT
kubectl apply -f  hhttps://raw.githubusercontent.com/donvito/learngo/master/mongo-microservice/mongodb/mongodb-deployment.yaml
kubectl get deployment mongodb
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
mongodb   1/1     1            1           4m43s

MONGODB SERVICE
kubectl apply -f  https://raw.githubusercontent.com/zencircle/minikube/master/mongodb-service.yaml

kubectl get svc mongodb
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
mongodb   NodePort   10.103.180.226   <none>        27017:32017/TCP   26m

Test DB is running 
root@mongodb-6f46795b8c-5mhmd:/# mongo --host localhost:27017

minikube service mongodb --url
http://192.168.99.100:32017

curl  http://192.168.99.100:32017
It looks like you are trying to access MongoDB over HTTP on the native driver port.
