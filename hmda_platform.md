## CI/CD in Kubernetes (local)

To build and run the application in Kubernetes (local development), the following steps must be taken:

1. Make sure that [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) is installed and configured for your system
2. Make sure that [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) is installed. When working in local development mode. `minikube` should have a sufficient resources to be able to run all the necessary containers for the `HMDA Platform`.
```
minikube start --cpus=4 --memory=8192
```
Make sure that `kubectl` is properly configured to point to `minikube`.  
```
kubect1 config user-context minikube
```   
3. Make sure that [Helm](https://helm.sh/) client and as well as Tiller, the server side component is installed
```
kubectl create -f https://raw.githubusercontent.com/istio/istio/master/install/kubernetes/helm/helm-service-account.yaml
helm init --service-account tiller
```
4. Install [Cassandra](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/). Cassandra is required to be running and correctly configured for hmda-platform
```
kubectl apply -f https://k8s.io/examples/application/cassandra/cassandra-service.yaml
kubectl apply -f https://k8s.io/examples/application/cassandra/cassandra-statefulset.yaml
Note: If you running low on resources you could have a single instance
kubectl scale --replicas=1 statefulset cassandra
```
5. Add credentials for Cassandra

```shell
kubectl create secret generic cassandra-credentials --from-literal=cassandra.username=<username> --from-literal=cassandra.password=<password>
kubectl create secret generic cassandra-credentials --from-literal=cassandra.username=cassandra --from-literal=cassandra.password=cassandra
```
5. Add institution api credentails
```
kubectl create secret generic inst-postgres-credentials --from-literal=username=postgres --from-literal=password=postgres --from-literal=host=postgres --from-literal=url="jdbc:postgresql://postgresql:5432/hmda?user=hmda&password=password&sslmode=require&ssl=true&sslfactory=org.postgresql.ssl.NonValidatingFactory"
```
### HMDA-PLATFORM
6. Download git repo
```
git clone https://github.com/cfpb/hmda-platform/
cd hmda-platform

If required, sync with a remote Git repository
git pull origin master
```
7. Add config-maps required to hmda-platform 
```
kubectl apply -f kubernetes/config-maps/minikube
```
8. Update the CPU/Memory values (minikube resources are limited)
Edit `kubernetes/hmda-platform/templates/deployment.yaml` file to mininum values
```
grep -A6 resources kubernetes/hmda-platform/templates/deployment.yaml 
        resources:
          limits:
            memory: "256Mi"
            cpu: "0.2"
          requests:
            memory: "64Mi"
            cpu: "0.1"
```
9. Delete affinity rules required only for production
Edit `kubernetes/hmda-platform/templates/deployment.yaml` file, remove below lines
```
      affinity:
       podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - "hmda-platform"
                - "keycloak"
            topologyKey: kubernetes.io/hostname
```
10. Install platform helm chart
```
helm upgrade --install --force \
--namespace=default \
--values=kubernetes/hmda-platform/values.yaml \
--set image.tag=latest \
--set service.name=hmda-platform-api \
hmda-platform \
```
* Install Ambassador Helm Chart
```
#helm upgrade --install --wait ambassador datawire/ambassador
kubectl apply -f https://raw.githubusercontent.com/zencircle/minikube/master/ambassador/deployment.yaml 
kubectl apply -f https://raw.githubusercontent.com/zencircle/minikube/master/ambassador/service.yaml
```
* Verify hmda-platform cluster is running endpoints 
```
minikube service ambassador  --url
http://192.168.99.103:30100
```
http://192.168.99.103:30100/v2/cluster/
http://192.168.99.103:30100/v2/public/
http://192.168.99.103:30100/v2/filing/
http://192.168.99.103:30100/v2/admin/

* hmda-platform application allows filing for authenticated users, keycloak is the authentication/authorization service. Keycloak storeds it information on postgresql 
### Keycloak

Postresql

Make sure the two secrets are created: `realm` from the file under `/kubernetes/keycloak`, and `keycloak-credentials`
with the key `password` set to the Postgres password.  Find the URL of the Postgres database, and then install Keycloak with 
this command:

```bash
helm upgrade -i -f kubernetes/keycloak/values.yaml keycloak stable/keycloak --set keycloak.persistence.dbHost="<db URL>"
```

### Install institutions-api

The institutions-api chart has two secret dependencies: `cassandra-credentials` (which is also needed by the hmda-platform)
and `inst-postgres-credentials`.  These keys need to be created if they don't already exist.  
* Cassandra secret keys: `cassandra.username` and `cassandra.password` 
* InstApi secret keys: `host`, `username` and `password`

If running locally, the Institutions API must be pointed at a local instance of Cassandra.  This can be done in the install command:
```bash
helm upgrade -i -f kubernetes/institutions-api/values.yaml institutions-api ./kubernetes/institutions-api/ --set cassandra.hosts="<Docker IP>"
```
If deploying to HMDA4, run the above command without the `set` flag and it will connect automatically.

If deploying and pointing to a new database, run with the flag `--set postgres.create-schema="true"`


```
### Install hmda-platform
```bash
helm upgrade --install --force --namespace=default \
--values=kubernetes/hmda-platform/values.yaml 
--set image.tag=latest 
--set service.name=hmda-platform-api 
--set image.pullPolicy=Always \
hmda-platform \
kubernetes/hmda-platform
```

