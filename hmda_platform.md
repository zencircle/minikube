## HMDA Platform Kubernetes (local) Deployment
### Pre-requisites
To build and run the application in Kubernetes (local development), the following steps must be taken:

1. Make sure that [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) is installed and configured for your system
2. Make sure that [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) is installed. When working in local development mode. `minikube` should have a sufficient resources to be able to run all the necessary containers for the `HMDA Platform`.
```
minikube start --cpus=4 --memory=8192
```
4. Make sure that `kubectl` is properly configured to point to `minikube`.  
```
kubect1 config user-context minikube
```   
5. Make sure that [Helm](https://helm.sh/) client and as well as Tiller, the server side component is installed
```
kubectl create -f https://raw.githubusercontent.com/istio/istio/master/install/kubernetes/helm/helm-service-account.yaml
helm init --service-account tiller
```
### Install [Cassandra](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/). 
Cassandra is required to be running and configured for hmda-platform
```
kubectl apply -f https://k8s.io/examples/application/cassandra/cassandra-service.yaml
kubectl apply -f https://k8s.io/examples/application/cassandra/cassandra-statefulset.yaml
```
### Install HMDA-PLATFORM
1. Add credentials for Cassandra

```
kubectl create secret generic cassandra-credentials --from-literal=cassandra.username=cassandra --from-literal=cassandra.password=cassandra
```
2. Add institution api credentails
```
kubectl create secret generic inst-postgres-credentials --from-literal=username=postgres --from-literal=password=postgres --from-literal=host=postgres --from-literal=url="jdbc:postgresql://postgresql:5432/hmda?user=hmda&password=postgres&sslmode=require&ssl=true&sslfactory=org.postgresql.ssl.NonValidatingFactory"
```
6. Download git repo
```
git clone https://github.com/cfpb/hmda-platform/
cd hmda-platform

If required, sync with a remote Git repository
git pull origin master
```
7. Add config-maps required to hmda-platform 
```
kubectl apply -f kubernetes/config-maps/
```    
Update `kubectl edit configmap cassandra-configmap` cassandra-hosts value to `cassandra`      
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
10. Install hmda-platform helm chart
```
helm upgrade --install --force \
--namespace=default \
--values=kubernetes/hmda-platform/values.yaml \
--set image.tag=latest \
--set service.name=hmda-platform-api \
hmda-platform \
```
11. Install **[Ambassador](https://www.getambassador.io/user-guide/getting-started/)** API Gateway
```
kubectl apply -f https://raw.githubusercontent.com/zencircle/minikube/master/ambassador/deployment.yaml 
kubectl apply -f https://raw.githubusercontent.com/zencircle/minikube/master/ambassador/service.yaml
```
Ambassador diagnostics page : http://192.168.99.103:30101/ambassador/v0/diag/     
12. Verify hmda-platform cluster is running by checking endpoint
```
minikube service ambassador  --url
```
http://192.168.99.103:30100   
http://192.168.99.103:30100/v2/cluster/    
### Install hmda-platform-ui, it is frontend web-service to hmda-platform
1. Download git repo
```
git clone https://github.com/cfpb/hmda-platform-ui
cd hmda-platform-ui/
```
2. Update `kubernetes/hmda-platform-ui/values.yaml` file since resources on minikube are limited
```
grep -A6 resources kubernetes/hmda-platform-ui/values.yaml 
resources:
  limits:
    cpu: 10m
    memory: 24Mi
  requests:
    cpu: 10m
    memory: 24i
```
3. Install hmda-platform-ui helm chart
```
helm upgrade --install --force \
--values=kubernetes/hmda-platform-ui/values.yaml \
--set image.tag=latest \
hmda-platform-ui \
kubernetes/hmda-platform-ui
```
4. Frontend URL 
http://192.168.99.103:30100/filing/2018/
### Keycloak

1. Install [Postresql](https://github.com/helm/charts/tree/master/stable/postgresql) from helm chart
Note: To similplify the configuraiton we will be sharing postgresql instance between keycloak and instituions-api
```
helm install --name postgresql \
  --set postgresqlUsername=postgres,postgresqlPassword=postgres,postgresqlDatabase=hmda \
    stable/postgresql
```
Make sure the two secrets are created: `realm` from the file under `/kubernetes/keycloak`, and `keycloak-credentials`
with the key `password` set to the Postgres password.  Find the URL of the Postgres database, and then install Keycloak with 
this command:

2. Update the CPU/Memory values (minikube resources are limited)
Edit `kubernetes/keycloak/values.yaml` file to mininum values
```
grep -A6 resources kubernetes/keycloak/values.yaml
  resources:
     limits:
       cpu: "100m"
       memory: "1024Mi"
     requests:
       cpu: "100m"
       memory: "1024Mi"
```
3. Delete affinity rules required only for production
Edit `kubernetes/keycloak/values.yaml` file, remove below lines
```
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
Keycloak helm chart installation
Add secerts and configs
```
kubectl create secret generic keycloak-credentials --from-literal=password=postgres
kubectl create secret generic realm-secret --from-file=kubernetes/keycloak/realm.json
kubectl apply -f kubernetes/keycloak/keycloak-https.yaml
kubectl create -f kubernetes/keycloak/keycloak-ambassador.yaml 
```
helm chart
```
helm upgrade --install --force --namespace=default \
--values=kubernetes/keycloak/values.yaml \
--set keycloak.persistence.dbHost=postgresql-postgresql \
--set keycloak.persistence.dbName=hmda \
--set keycloak.username=keycloak \
--set keycloak.password=keycloak \
keycloak stable/keycloak
```
Manually Update Keycloak script, correct path "/opt/jboss/docker-entrypoint.sh", this fixes error
`/scripts/keycloak.sh: line 46: /opt/jboss/tools/docker-entrypoint.sh: No such file or directory`
kubectl edit configmap keycloak

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
