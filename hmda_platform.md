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
4. Install [Cassandra](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/)
```
kubectl apply -f https://k8s.io/examples/application/cassandra/cassandra-service.yaml
kubectl apply -f https://k8s.io/examples/application/cassandra/cassandra-statefulset.yaml
```
Add credentials for Cassandra

```shell
kubectl create secret generic cassandra-credentials --from-literal=cassandra.username=<username> --from-literal=cassandra.password=<password>
kubectl create secret generic cassandra-credentials --from-literal=cassandra.username=cassandra --from-literal=cassandra.password=cassandra
```

HMDA-PLATFORM
```
git clone https://github.com/cfpb/hmda-platform/
cd hmda-platform

If required, sync with a remote Git repository
git pull origin master

Update the CPU/Memory values (minikube resources are limited)
grep -A6 resources kubernetes/hmda-platform/templates/deployment.yaml 
        resources:
          limits:
            memory: "256Mi"
            cpu: "0.2"
          requests:
            memory: "64Mi"
            cpu: "0.1"

helm upgrade --install --force \
--namespace=default \
--values=kubernetes/hmda-platform/values.yaml \
--set image.tag=latest \
--set service.name=hmda-platform-api \
hmda-platform \
```
5. Install the `Jenkins` Helm Chart, as follows:

* Create namespace for `Jenkins`: 

```bash
kubectl apply -f kubernetes/jenkins-namespace.yaml
```

* Add Ambassador Helm Repository

```shell
helm repo add datawire https://www.getambassador.io
```

* Install Ambassador Helm Chart

```shell
helm upgrade --install --wait ambassador datawire/ambassador
```

* Create Persistent Volume for Jenkins


* Install Jenkins Chart

```shell
helm install --name jenkins -f kubernetes/jenkins-values.yaml stable/jenkins --namespace jenkins-system
```

You can access `Jenkins` by issuing `minikube service --n jenkins-system jenkins` and logging in with `admin/admin`.

Follow the on screen instructions to finalize `Jenkins` setup. When logged in, update plugins if necessary.

* Docker Hub Credentials

Add credentials in Jenkins for `Docker Hub` so that images can be pushed as part of `Jenkins` pipeline builds.

### Install Keycloak

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

### Install modified-lar
```bash
helm upgrade --install --force --namespace=default \
 --values=kubernetes/modified-lar/values.yaml \
 --set image.tag=latest \
 --set image.pullPolicy=Always \
modified-lar \
kubernetes/modified-lar
```
### Install census-api
```bash
helm upgrade --install --force \
--namespace=default \
--values=kubernetes/census-api/values.yaml \
--set image.tag=latest \
--set image.pullPolicy=Always \
census-api \
kubernetes/census-api
```
### Install hmda-regulator
```bash
helm upgrade --install --force --namespace=default \
--values=kubernetes/hmda-regulator/values.yaml \
--set image.tag=latest \
--set image.pullPolicy=Always \
hmda-regulator \
kubernetes/hmda-regulator
```
### Install hmda-analytics
```bash
helm upgrade --install --force --namespace=default \
--values=kubernetes/hmda-analytics/values.yaml \
--set image.tag=latest \
--set image.pullPolicy=Always \
hmda-analytics \
kubernetes/hmda-analytics
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
### Install check-digit
```bash
helm upgrade --install --force --namespace=default \
--values=kubernetes/check-digit/values.yaml \
--set image.tag=master \
check-digit \
kubernetes/check-digit
```
### Install Institutions API
6. OPTIONAL: Install [Istio](https://istio.io/) Service Mesh

* Install Istio with Helm. Download the Istio distribution and run from the Istio root path:

```bash
helm install install/kubernetes/helm/istio --name istio --namespace istio-system
```

* Make sure automatic sidecar injection is supported: 

```bash
kubectl api-versions | grep admissionregistration
```

* Enable automatic sidecar injection in the `default` namespace: 

```bash
kubectl label namespace default istio-injection=enabled
``` 

To check that this operation succeeded: 

```bash
kubectl get namespace -L istio-injection
```
