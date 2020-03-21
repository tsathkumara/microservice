# Table of contents
- [Table of contents](#table-of-contents)
  - [Run PostgreSQL database locally as docker container <a name="database"></a>](#run-postgresql-database-locally-as-docker-container-a-name%22database%22a)
  - [Getting started with a helm chart deployment <a name="dummy-helm-chart-deployment"></a>](#getting-started-with-a-helm-chart-deployment-a-name%22dummy-helm-chart-deployment%22a)
  - [Backend - Python Flask <a name="backend"></a>](#backend---python-flask-a-name%22backend%22a)
      - [Overview of backend env. variables <a name="env-variables"></a>](#overview-of-backend-env-variables-a-name%22env-variables%22a)
      - [Run backend locally <a name="run-backend-locally"></a>](#run-backend-locally-a-name%22run-backend-locally%22a)
      - [Run backend as docker container locally <a name="run-backend-as-docker-container-locally"></a>](#run-backend-as-docker-container-locally-a-name%22run-backend-as-docker-container-locally%22a)
      - [Backend helm chart deployment <a name="backend-helm-chart-deployment"></a>](#backend-helm-chart-deployment-a-name%22backend-helm-chart-deployment%22a)
      - [Get inside busybox and call your flask instance](#get-inside-busybox-and-call-your-flask-instance)
      - [Get inside python instance POD and call your flask instance](#get-inside-python-instance-pod-and-call-your-flask-instance)
      - [Scale up/down your back-end app deployment](#scale-updown-your-back-end-app-deployment)
      - [Render templates files](#render-templates-files)
  - [Frontend - React app <a name="frontend"></a>](#frontend---react-app-a-name%22frontend%22a)
      - [Run frontend locally <a name="run-frontend-locally"></a>](#run-frontend-locally-a-name%22run-frontend-locally%22a)
      - [Run frontend as docker container locally <a name="run-frontend-as-docker-container-locally"></a>](#run-frontend-as-docker-container-locally-a-name%22run-frontend-as-docker-container-locally%22a)
      - [Frontend helm chart deployment <a name="frontend-helm-chart-deployment"></a>](#frontend-helm-chart-deployment-a-name%22frontend-helm-chart-deployment%22a)
      - [Scale up/down your front-end app deployment](#scale-updown-your-front-end-app-deployment)
  - [Nginx Controller Proxy <a name="nginx-controller-proxy"></a>](#nginx-controller-proxy-a-name%22nginx-controller-proxy%22a)
  - [Create helm chart repository at your Github repository](#create-helm-chart-repository-at-your-github-repository)
  - [Deploy micro-backend and micro-frontend helm chart from Github](#deploy-micro-backend-and-micro-frontend-helm-chart-from-github)
  - [Create helm chart repository based on Chartmuseum helm chart](#create-helm-chart-repository-based-on-chartmuseum-helm-chart)
  - [Deploy micro-backend and micro-frontend helm chart from Chartmuseum](#deploy-micro-backend-and-micro-frontend-helm-chart-from-chartmuseum)
  - [Troubleshooting section <a name="troubleshooting"></a>](#troubleshooting-section-a-name%22troubleshooting%22a)
    - [If helm chart has to be renamed from foo to bar](#if-helm-chart-has-to-be-renamed-from-foo-to-bar)


## Run PostgreSQL database locally as docker container <a name="database"></a>

```bash
# Clean all docker images/processes/... if neceassary
docker rmi $(docker images -q) -f
docker ps --filter status=dead --filter status=exited -aq | xargs -r docker rm -v

# Start postgress as docker instance
docker run --name micro-postgres -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=password -p 5432:5432 -d postgres:alpine

# Connect to PostgreSQL from your laptop
psql --host=localhost --port=5432 -U postgres

# Create database, user and grant privileges
CREATE DATABASE  microservice;
CREATE USER micro WITH ENCRYPTED PASSWORD 'password'; 
GRANT ALL PRIVILEGES ON DATABASE microservice TO micro;
ALTER DATABASE microservice OWNER TO micro;

# Connect to databse and check request_ips table
psql --host=localhost --port=5432 -U micro -d microservice
select * from request_ips;
```

## Getting started with a helm chart deployment <a name="dummy-helm-chart-deployment"></a>

If there is someone who does not know anything about <br>
helm charts there is a simple deployment available <br>
Dummy Dokuwiki deployment by using helm chart

```bash
# Deploy Dokuwiki with helm v2
helm install \
--name dw \
--set service.type=NodePort \
--set service.nodePorts.http=30111 \
--set persistence.enabled=false \
--set dokuwikiUsername=admin,dokuwikiPassword=password \
stable/dokuwiki \
--tls

# Deploy Dokuwiki with helm v3
helm3 install \
dw \
--set service.type=NodePort \
--set service.nodePorts.http=30111 \
--set persistence.enabled=false \
--set dokuwikiUsername=admin,dokuwikiPassword=password \
stable/dokuwiki \
```


## Backend - Python Flask <a name="backend"></a>

#### Overview of backend env. variables <a name="env-variables"></a>

Following environmental variables are used inside docker image.

```bash
PSQL_DB_USER        default='micro'
PSQL_DB_PASS        default='password'
PSQL_DB_NAME        default='microservice'
PSQL_DB_ADDRESS     default='127.0.0.1'
PSQL_DB_PORT        default='5432'
```

#### Run backend locally <a name="run-backend-locally"></a>

```bash
# Create python virtualenv
python3 -m venv venv_micro

# Activate virtualenv 
source venv_micro/bin/activate

# Install requirements from requirements.txt
pip install -r requirements.txt

# Start Flask in non-production mode
cd ../backend
export FLASK_APP=app
flask run
```

#### Run backend as docker container locally <a name="run-backend-as-docker-container-locally"></a>

```bash
# Stop and remove docker container process if previously started
docker stop micro-service && docker rm micro-service || :

# Build docker image
docker build -t <account>/microservice:v0.0.1 .

# Run docker container locally
docker run \
--rm \
--name micro-service \
-it \
-e PSQL_DB_ADDRESS=192.168.1.45 \
-p 5001:8000 \
-d <account>/microservice:v0.0.1

# Get inside docker container
docker exec -it micro-service sh

# Push docker image to public docker registry
docker login
docker push <account>/microservice:v0.0.1
```

#### Install helm v3

```bash
function install_helm3 () {
  mkdir /tmp/helm3_unpacked
  curl --output /tmp/helm3.tgz -L https://get.helm.sh/helm-v3.1.1-linux-amd64.tar.gz
  tar -xvf /tmp/helm3.tgz  -C /tmp/helm3_unpacked
  cp /tmp/helm3_unpacked/linux-amd64/helm /usr/bin/helm3
  chmod +x /usr/bin/helm3
  helm3
}

install_helm3

# In case you have no helm chart repository added
helm3 repo \
add stable https://kubernetes-charts.storage.googleapis.com/

# Verify your helm chart repository repo helm v3
helm3 repo list
```


#### Backend helm chart deployment <a name="backend-helm-chart-deployment"></a>

Backend (Python Flask) helm chart (micro-backend) is dependent on a database PostgreSQL.<br>
There are two options how one can include database (postgresql) as dependency for this helm chart:<br>
backend helm chart.

1) create **requirements.yaml** file inside **micro-backend**

```bash
# Create requirements.yaml file
cat <<EOF > requirements.yaml
dependencies:
- name: postgresql
  repository: https://kubernetes-charts.storage.googleapis.com
  version: 3.18.3
EOF

# Update dependencies
helm dependency update
```

2) download **postgresql** helm chart by using `helm fetch <repo>/<chart-name>` <br>and copy it to **charts/** folder inside your helm chart (micro-backend)

```bash
helm fetch stable/postgresql
cp postgresql*.tgz charts/
```

```bash
# Before the very first deployment 
cd helm-charts/micro-backend 
# For helm v2
helm dependency update
# For helm v3
helm3 dependency update

cd charts/
tar -xvzf postgresql-3.18.3.tgz
rm postgresql-3.18.3.tgz -rf && cd ..
sed -i 's/apiVersion: apps\/v1beta2/apiVersion: apps\/v1/g' charts/postgresql/templates/statefulset.yaml charts/postgresql/templates/statefulset-slaves.yaml

# Deploy backend helm chart with helm v2
helm install \
--name backend \
--set service.type=NodePort \
--set service.nodePort=30222 \
helm-charts/micro-backend \
--tls

# Deploy backend helm chart with helm v3
helm3 install \
backend \
--set service.type=NodePort \
--set service.nodePort=30222 \
helm-charts/micro-backend 


# Describe backend pod
kubectl describe pod \
$(kubectl get pods | grep backend-micro-backend | awk -F" " {'print $1'})

# See what is going on in logs
kubectl logs -f \
$(kubectl get pods | grep backend-micro-backend | awk -F" " {'print $1'})

# Update already deployed helm chart with helm v2
helm upgrade backend helm-charts/micro-backend --tls

# Update already deployed helm chart with helm v3
helm3 upgrade backend \
--set replicaCount=4 \
--set service.type=NodePort  \
--set service.nodePort=30222 \
helm-charts/micro-backend 



# Delete helm chart deployment with helm v2
helm delete --purge backend --tls

# Delete helm chart deployment with helm v3
helm3 delete backend 

```

Verify your backend deployment via:

```bash
curl http://<ip_address>:30222/api/saveip
```

#### Get inside busybox and call your flask instance

```bash
# get inside your busybox
kubectl exec -it busybox -- sh

uname -n
ip a
wget -O - http://backend-micro-backend:80/api/isalive
wget -O - http://backend-micro-backend:80/api/getallips
wget -O - http://backend-micro-backend:80/api/saveip
wget -O - http://backend-micro-backend:80/api/getallips
```

#### Get inside python instance POD and call your flask instance

```bash
kubectl exec -it backend-micro-backend-7d887bb858-sd9hb -- sh

ps -ef 
PID   USER     TIME  COMMAND
1  root       7:08 {gunicorn} /usr/local/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8000
8  root      18:42 {gunicorn} /usr/local/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8000
23 root      0:00 sh

wget -O - http://127.0.0.1:8000/api/isalive
wget -O - http://127.0.0.1:8000/api/saveip
wget -O - http://127.0.0.1:8000/api/getallips
```


#### Scale up/down your back-end app deployment

```bash
# Describe micro-backend svc
kubectl \
describe \
svc \
$(kubectl get svc | grep micro-backend | awk -F" " '{print $1}')

# Check number of micro-backend pods
kubectl get pods -o wide| grep micro-backend

# Scale up your back-end deployment to rs=3 with helm v2
helm upgrade backend \
--set replicaCount=3 \
--set service.nodePort= \
helm-charts/micro-backend \
--tls

# Scale up your back-end deployment to rs=3 with helm v3
helm3 upgrade backend \
--set replicaCount=3 \
--set service.nodePort= \
helm-charts/micro-backend 
```

#### Render templates files

```bash
# If want to see how supplied values will be rendered before deployment
# with helm v2
/opt/microservice
helm template -x templates/service.yaml helm-charts/micro-backend
helm template -x templates/ingress.yaml helm-charts/micro-backend 
helm template -x templates/deployment.yaml helm-charts/micro-backend

# If want to see how supplied values will be rendered before deployment
# with helm v3
/opt/microservice
helm3 template -x templates/service.yaml helm-charts/micro-backend
helm3 template -x templates/ingress.yaml helm-charts/micro-backend 
helm3 template -x templates/deployment.yaml helm-charts/micro-backend
```

## Frontend - React app <a name="frontend"></a>

#### Run frontend locally <a name="run-frontend-locally"></a>

Please execute following lines when running React frontend app<br>
locally (e.g. at your laptop)

```bash
cd ../frontend/
npm install
# npm audit fix --force
npm start
npm run build
```

#### Run frontend as docker container locally <a name="run-frontend-as-docker-container-locally"></a>

```bash
cd frontend
# Build frontend docker image
docker build -t <account>/frontend:v0.0.3 .

# Run front-end React as docker container locally
docker run --rm --name ft -it -p 3001:80 -d <account>/frontend:v0.0.3

# Get inside docker container
docker exec -it ft sh

# Push docker image to public docker registry
docker login
docker push <account>/frontend:v0.0.3
```

#### Frontend helm chart deployment <a name="frontend-helm-chart-deployment"></a>

If want to quickly expose frontend app as service type NodePort<br>
to be able to access it immediately - please use following command:


```bash
# Install frontend helm chart with helm v2
helm install \
--name frontend \
--set service.type=NodePort \
--set service.nodePort=30333 \
helm-charts/micro-frontend \
--tls

# Deploy micro-backend (in 4 replicas) helm chart with helm v3 together with micro-frontend
helm3 install backend \
--set replicaCount=4 \
--set service.type=NodePort  \
--set service.nodePort=30222 \
helm-charts/micro-backend 

# Install frontend helm chart with helm v3
helm3 install \
frontend \
--set service.type=NodePort \
--set service.nodePort=30333 \
helm-charts/micro-frontend 


# Describe frontend pod
kubectl describe pod \
$(kubectl get pods | grep frontend-micro-frontend | awk -F" " {'print $1'})

# See frontend pod logs fron docker container
kubectl logs -f $(kubectl get pods | grep frontend-micro-frontend | awk -F" " {'print $1'})

# Update already deployed helm chart with helm v2
helm upgrade frontend helm-charts/micro-backend --tls

# Update already deployed helm chart with helm v3
helm3 upgrade frontend helm-charts/micro-backend 

# Delete helm chart deployment helm v2
helm delete --purge frontend --tls

# Delete helm chart deployment helm v3
helm3 delete frontend 
```

Verify your frontend deployment via:

```bash
curl http://<ip_address>:30333/app/
```

#### Scale up/down your front-end app deployment

```bash
# Describe micro-frontend svc
kubectl \
describe \
svc \
$(kubectl get svc | grep micro-frontend | awk -F" " '{print $1}')

# Check number of micro-frontend pods
kubectl get pods -o wide| grep micro-frontend

# Scale up your front-end deployment to rs=2 helm v2
helm upgrade frontend \
--set replicaCount=2 \
--set service.nodePort= \
helm-charts/micro-frontend \
--tls

# Scale up your front-end deployment to rs=2 helm v3
helm3 upgrade frontend \
--set replicaCount=2 \
--set service.nodePort= \
helm-charts/micro-frontend 
```



![](images/react-frontend.png)

## Nginx Controller Proxy <a name="nginx-controller-proxy"></a>

You must have an ingress controller to satisfy an Ingress. Only creating an Ingress resource has no effect. <br>
If want to avoid exposing **NodePort** service <br>
type for each app deployed in Kubenretes - please <br>
use following deployment:

```bash

# Remove NodePort from backend deployment helm v2
helm upgrade backend \
--set service.type=ClusterIP \
--set service.nodePort= \
helm-charts/micro-backend \
--tls

# Remove NodePort from backend deployment helm v3
helm3 upgrade backend \
--set service.type=ClusterIP \
--set service.nodePort= \
helm-charts/micro-backend 

# Remove NodePort from frontend deployment helm v2
helm upgrade frontend \
--set service.type=ClusterIP \
--set service.nodePort= \
helm-charts/micro-frontend \
--tls

# Remove NodePort from frontend deployment helm v3
helm3 upgrade frontend \
--set service.type=ClusterIP \
--set service.nodePort= \
helm-charts/micro-frontend 

# nginx-ingress deployment helm v2
helm install \
--name ingress \
--set controller.service.type=NodePort \
--set controller.service.nodePorts.http=30444 \
stable/nginx-ingress \
--tls

# In case you have no helm chart repository added
helm3 repo \
add stable https://kubernetes-charts.storage.googleapis.com/

# Verify your helm chart repository repo helm v3
helm3 repo list


# !!! Consideration 
dig k8s.<servername>.com
---
;; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> k8s.<servername>.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64244
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;k8s.<servername>.com.            IN      A

;; ANSWER SECTION:
k8s.<servername>.com.     53      IN      A       1.2.3.4

;; Query time: 11 msec
;; SERVER: 10.1.94.8#53(10.1.94.8)
;; ...
;; MSG SIZE  rcvd: 63
---


# nginx-ingress deployment helm v3
helm3 install \
ingress \
--set controller.service.type=NodePort \
--set controller.service.nodePorts.http=30444 \
stable/nginx-ingress 

# Explore nginx-ingress configuration
kubectl \
exec \
-it \
$(kubectl get pods | grep "nginx-ingress-controller" | awk -F" " '{print $1}')\
 -- cat /etc/nginx/nginx.conf > \
 /tmp/nginx-controller.conf

# Delete nginx ingress controller helm v2
helm delete --purge ingress --tls

# Delete nginx ingress controller helm v3
helm3 delete ingress 
```

## Create (helm v2) helm chart repository at your Github repository

```bash
git clone https://github.com/xjantoth/microservice.git
cd microservice
mkdir -p docs/charts/
cd helm-charts
helm package micro-backend
helm package micro-frontend
cp ../micro-backend-0.1.0.tgz docs/charts/
cp ../micro-frontend-0.1.0.tgz docs/charts/
helm repo index .
git add docs/charts/
git commit -m "Creating helm chart repository"
git push 
```
![](images/github-pages.png)

```bash
helm repo add course https://xjantoth.github.io/microservice/charts
helm search course/
```

## Create (helm v3) helm chart repository at your Github repository

```bash
git clone https://github.com/xjantoth/microservice.git
cd microservice

mkdir -p docs/hc-v3-repo
helm3 package helm-charts/micro-backend --destination docs/hc-v3-repo
helm3 package helm-charts/micro-frontend --destination docs/hc-v3-repo
helm3 repo index docs/hc-v3-repo

git add docs/hc-v3-repo
git commit -m "Creating helm v3 chart repository docs/hc-v3-repo"
git push 

helm3 repo add hc-v3-repo https://xjantoth.github.io/microservice/hc-v3-repo
helm3 repo update
helm3 repo list
helm3 search repo hc-v3-repo/
```


## Add one more dummy helm chart
```bash
cd microservice/docs/charts
helm create course
helm package course
helm repo index .
git add .
git commit -m "Adding course-..-.tgz helm chart to my Github repo."
git push

# Go to my master k8s server
helm repo update
helm search course/
```
## Deploy (Helm v2) micro-backend and micro-frontend helm chart from Github

```bash
helm install \
--name frontend \
--set service.type=ClusterIP \
--set service.nodePort= \
course/micro-frontend \
--tls

helm install \
--name backend \
--set service.type=ClusterIP \
--set service.nodePort= \
course/micro-backend \
--tls
```

## Deploy (Helm v3) micro-backend and micro-frontend helm chart from Github

```bash
# Add hc-v3-repo helm chart repo (Github based)
helm3 repo add hc-v3-repo https://xjantoth.github.io/microservice/hc-v3-repo

# In case you have no helm chart repository added
helm3 repo \
add stable https://kubernetes-charts.storage.googleapis.com/
helm3 repo update

helm3 install \
frontend \
--set service.type=ClusterIP \
--set service.nodePort= \
hc-v3-repo/micro-frontend

helm3 install \
backend \
--set replicaCount=4 \
--set service.type=ClusterIP \
--set service.nodePort= \
hc-v3-repo/micro-backend 

# nginx-ingress deployment helm v3
helm3 install \
ingress \
--set controller.service.type=NodePort \
--set controller.service.nodePorts.http=30444 \
stable/nginx-ingress 

# Result
helm3 ls
NAME    	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
backend 	default  	1       	2020-03-17 21:40:20.523605058 +0000 UTC	deployed	micro-backend-0.1.0 	1.0        
frontend	default  	1       	2020-03-17 21:37:54.086454881 +0000 UTC	deployed	micro-frontend-0.1.0	1.0        
ingress 	default  	1       	2020-03-17 21:41:23.573504347 +0000 UTC	deployed	nginx-ingress-1.34.1	0.30.0  
# Delete deployments

helm3 delete <deployment-name>
```

## Create helm chart repository based on Chartmuseum helm chart (Helm v2 & v3)

```bash
# (Helm v2) 
helm install \
--name chartmuseum \
--set persistence.pv.enabled=false \
--set env.open.DISABLE_API=false \
--set env.open.CONTEXT_PATH="/chartmuseum" \
--set ingress.enabled=true \
--set ingress.hosts[0].name="k8s.linuxinuse.com" \
--set ingress.hosts[0].path="/chartmuseum" \
stable/chartmuseum \
--tls \
--dry-run \
--debug

# (Helm v2) Add a new helm chart repository to your list with no authentication
helm repo list
helm repo add chartmuseum http://k8s.linuxinuse.com:30444/chartmuseum
helm repo update
helm search chartmuseum/

# (Helm v3) 
helm3 install \
chartmuseum \
--set persistence.pv.enabled=false \
--set env.open.DISABLE_API=false \
--set env.open.CONTEXT_PATH="/chartmuseum" \
--set ingress.enabled=true \
--set ingress.hosts[0].name="k8s.linuxinuse.com" \
--set ingress.hosts[0].path="/chartmuseum" \
stable/chartmuseum 


# (Helm v2) Upgrade your deployment with basic auth
helm upgrade \
chartmuseum \
--set persistence.pv.enabled=false \
--set env.open.DISABLE_API=false \
--set env.open.CONTEXT_PATH="/chartmuseum" \
--set ingress.enabled=true \
--set ingress.hosts[0].name="k8s.linuxinuse.com" \
--set ingress.hosts[0].path="/chartmuseum" \
--set env.secret.BASIC_AUTH_USER="user" \
--set env.secret.BASIC_AUTH_PASS="Start123#" \
stable/chartmuseum \
--tls \
--dry-run \
--debug

# (Helm v3) Upgrade your deployment with basic auth
helm3 upgrade \
chartmuseum \
--set persistence.pv.enabled=false \
--set env.open.DISABLE_API=false \
--set env.open.CONTEXT_PATH="/chartmuseum" \
--set ingress.enabled=true \
--set ingress.hosts[0].name="k8s.linuxinuse.com" \
--set ingress.hosts[0].path="/chartmuseum" \
--set env.secret.BASIC_AUTH_USER="user" \
--set env.secret.BASIC_AUTH_PASS="Start123#" \
stable/chartmuseum 


# (Helm v2) Add Chartmuseum to the list of available helm chart repsitories with authentication
helm repo add k8s http://k8s.linuxinuse.com:30444/chartmuseum --username user --password Start123#

# (Helm v3) Add Chartmuseum to the list of available helm chart repsitories
helm3 repo add k8s http://k8s.linuxinuse.com:30444/chartmuseum --username user --password Start123#


# (Helm v2) Package your helm charts
helm package helm-charts/micro-backend
helm package helm-charts/micro-frontend

# (Helm v3) Package your helm charts
helm3 package helm-charts/micro-backend
helm3 package helm-charts/micro-frontend

# (Helm v2) Push helm chart to Chartmuseum with no authentication
curl --data-binary "@micro-backend-0.1.0.tgz" http://k8s.linuxinuse.com:30444/chartmuseum/api/charts
{"saved":true}
curl --data-binary "@micro-frontend-0.1.0.tgz" http://k8s.linuxinuse.com:30444/chartmuseum/api/charts
{"saved":true}

# (Helm v2) Verify that your chart was saved in Chartmuseum
helm search chartmuseum/
NAME                            CHART VERSION   APP VERSION     DESCRIPTION             
chartmuseum/micro-backend       0.1.0           1.0             Backend Python Flask app

helm fetch chartmuseum/micro-backend

# Delete (Helm v2) Chartmuseum deployment 
helm delete chartmuseum --tls --purge

# (Helm v3) Push helm chart to Chartmuseum with authentication
curl -u user --data-binary "@micro-backend-0.1.0.tgz" http://k8s.linuxinuse.com:30444/chartmuseum/api/charts
{"saved":true}
curl -u user --data-binary "@micro-frontend-0.1.0.tgz" http://k8s.linuxinuse.com:30444/chartmuseum/api/charts
{"saved":true}

# Delete (Helm v3) Chartmuseum deployment 
helm3 delete chartmuseum 

```

## (Helm v2) Deploy micro-backend and micro-frontend helm charts from Chartmuseum 

```bash
# nginx-ingress deployment helm v2
helm install \
--name ingress \
--set controller.service.type=NodePort \
--set controller.service.nodePorts.http=30444 \
stable/nginx-ingress 

# micro-frontend deployment from Chartmuseum helm v2
helm install \
--name frontend \
--set service.type=ClusterIP \
--set service.nodePort= \
chartmuseum/micro-frontend \
--tls

# micro-backend deployment from Chartmuseum helm v3
helm install \
--name backend \
--set service.type=ClusterIP \
--set service.nodePort= \
chartmuseum/micro-backend \
--tls
```

## (Helm v3) Deploy micro-backend and micro-frontend helm charts from Chartmuseum
```bash
# nginx-ingress deployment helm v3
helm3 install \
ingress \
--set controller.service.type=NodePort \
--set controller.service.nodePorts.http=30444 \
stable/nginx-ingress 

# micro-frontend deployment from Chartmuseum helm v3
helm3 install \
frontend \
--set service.type=ClusterIP \
--set service.nodePort= \
k8s/micro-frontend 

# micro-backend deployment from Chartmuseum helm v3
helm3 install \
backend \
--set service.type=ClusterIP \
--set service.nodePort= \
k8s/micro-backend

# scale up micro-backend to 4 replicas (pods) helm v3
helm3 upgrade backend \
--set replicaCount=4 \
--set service.type=ClusterIP \
--set service.nodePort= \
k8s/micro-backend
```

## Troubleshooting section <a name="troubleshooting"></a>

There are plenty of situations when one needs to troubleshoot network connections between apps running inside pods and docker containers.

It is very useful to start a **busybox** pod and exec to this running pod:

```bash

kubectl apply -f https://k8s.io/examples/admin/dns/busybox.yaml

# Verify that busybox pod has been started
[root@k8s-master ~]# kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   92         3d19h

# Exec to busybox pod
root@k8s-master ~]# kubectl exec -it busybox -- sh 
/ # 
/ # uname -n 
busybox
/ # ip a | grep -E "inet "
    inet 127.0.0.1/8 scope host lo
    inet 192.168.0.66/24 brd 192.168.0.255 scope global eth0

```

Well now one get inside the pod which is on the same network as the rest of the pods. 

The **advantage** it that inside the pod there are commands like:

* wget
* telnet
* netstat

available.

### If helm chart has to be renamed from foo to bar

```bash
find . -type f -not -path '*/\.*' -exec sed -i 's/micro-chart/micro-backend/g' {} +

```

## USeeful links
https://forum.kde.org/viewtopic.php?f=270&t=154151
