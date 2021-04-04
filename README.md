# Packaging Applications with Helm for Kubernetes
Ressources for "Packaging Applications with Helm for Kubernetes"(Helm version 3)

## Creating lab environment
virtualbox installation
```
https://www.virtualbox.org/wiki/Downloads
```

minikube installation
```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
sudo mkdir -p /usr/local/bin/
sudo install minikube /usr/local/bin/
minikube version --short
minikube start --driver=hyperkit 
minikube addons enable ingress
minikube addons enable dashboard
```

edit /etc/hosts file
```bash
minikube ip
sudo vim /etc/hosts
{minikube ip} frontend.minikube.local
{minikube ip} backend.minikube.local
```

kubectl installation
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --short
minikube status
```

installing Helm
```bash
Download your [desired version](https://github.com/helm/helm/releases)
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version --short
``` 

check the current configuration of kubectl with one cluster
```bash
kubectl config view
```
## Playing with helm chart

adding a repo to helm and installing a chart from that repo
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm install my-release bitnami/nginx
```

upgrading a chart
```bash
helm upgrade my-release bitnami/nginx
```

removing a chart
```bash
helm uninstall my-release
```

There are more charts. Check this helm hub out
you can install third party apps quickly
```
https://artifacthub.io/
```

Building helm charts
the files are in this repo
```bash
https://github.com/gulec2000/helm3
https://github.com/gulec2000/helm3/tree/master/lab5_helm_chart_version1_final/chart/guestbook
```

Creating helm chart for an app
```bash
mkdir guestbook && cd guestbook
vim Chart.yaml
mkdir templates
vim frontend-service.yaml
vim frontend.yaml
vim ingress.yaml
helm install demo-guestbook guestbook
helm list --short
helm get manifest demo-guestbook | less
```

Change the version of the app
```text
edit version of the app in Chart.yaml and image version in frontend.yaml
```

```bash
helm upgrade demo-guestbook guestbook
browse http://frontend.minikube.local
helm status demo-guestbook
```

rollback to version 1.0
```bash
helm rollback demo-guestbook 1
helm history demo-guestbook
```

Building an umbrella helm chart and version 2.0
```bash
mkdir guestbook && cd guestbook
touch Chart.yaml
mkdir chart && cd chart
mkdir frontend database backend
cd frontend &% mkdir templates && vim Chart.yaml
cd templates & touch frontend-service.yaml && touch frontend.yaml && touch frontend-configmap.yaml && touch ingress.yaml
cd ../../backend && mkdir templates && vim Chart.yaml
cd templates & touch backend-service.yaml && touch backend.yaml && touch backend-secret.yaml
cd ../../database && mkdir templates && vim Chart.yaml
cd templates & touch mongodb-persistent-volume.yaml && touch mongodb.yaml && touch mongodb-persistent-volume-claim.yaml \
&& touch mongodb-secret.yaml && touch mongodb-service.yaml
helm upgrade demo-guestbook guestbook
browse http://frontend.minikube.local
helm status demo-guestbook 
```

helm template testing
```bash
helm template [chart name] or
helm install [release name] [chart name] --dry-run --debug
```

## Customizing frontend chart and using values.yaml file to be able to create a customizable chart.

editing frontend-configmap.yaml file
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}-config
data:
  guestbook-name: {{ .Values.config.guestbook_name }}
  backend-uri: {{ .Values.config.backend_uri }}
```

creating values.yaml in frontend chart
```yaml
config:
  guestbook_name: "MyPopRock Festival 2.0"
  backend_uri: "http://backend.minikube.local/guestbook"
image:
  repository: phico/frontend
  tag: "2.0"
replicaCount: 1
service:
  type: ClusterIP
  port: 80
ingress:
  host: frontend.minikube.local
```

editing frontend.yaml file
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }} 
  selector:
    matchLabels:
      app: {{ .Release.Name }}-{{ .Chart.Name }} 
  template:
    metadata: 
      labels:
        app: {{ .Release.Name }}-{{ .Chart.Name }}
    spec:
      containers:
      - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: Always
        name: {{ .Release.Name }}-{{ .Chart.Name }}
        ports:
        - name: http
          containerPort: 4200
        env:
        - name: BACKEND_URI
          valueFrom:
            configMapKeyRef:
              name: {{ .Release.Name }}-{{ .Chart.Name }}-config
              key: backend-uri
        - name: GUESTBOOK_NAME
          valueFrom:
            configMapKeyRef:
              name: {{ .Release.Name }}-{{ .Chart.Name }}-config
              key: guestbook-name
```

editing frontend-service.yaml file
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: {{ .Release.Name }}-{{ .Chart.Name }}
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - protocol: "TCP"
      port: {{ .Values.service.port }}
      targetPort: 4200
  selector:
    app: {{ .Release.Name }}-{{ .Chart.Name }}
```

editing ingress.yaml file
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}-ingress
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        backend:
          serviceName: {{ .Release.Name }}-{{ .Chart.Name }}
          servicePort: 80
```
The same changes apply for backend and database charts

## Creating a customizable chart using global Chart.yaml, values.yaml and local _helpers.tpl files with umbrella helm chart topology

adding _helper.tpl file for each template in order to create customizable templates using logic
for frontend
```tpl
{{- define "frontend.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
```
for database
```tpl
{{- define "database.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
```
for backend
```tpl
{{- define "backend.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
```

changing the yaml file in frontend,database and backend according to _helper.tpl file
```text
frontend: replace {{ .Release.Name }}-{{ .Chart.Name }} into {{ include "frontend.fullname" . }}
backend: replace {{ .Release.Name }}-{{ .Chart.Name }} into {{ include "backend.fullname" . }}
database: replace {{ .Release.Name }}-{{ .Chart.Name }} into {{ include "database.fullname" . }}
```

## Solving the problem backend not able to accss database due to naming issue of mongodb. this depends on release name
adding mongodb_uri credentials of backend into values.yaml in backend
```yaml
secret:
  mongodb_uri:
    username: your_db_username
    password: your_db_password
    dbchart: database
    dbport: 27017
    dbconn: "guestbook?authSource=admin"
```

changing data.mongodb_uri part of backend-secret.yam file accordingly. This helps building the name dynamically
```yaml
data:
  mongodb-uri: {{ with .Values.secret.mongodb_uri -}}
  {{- list "mongodb://" .username ":" .password "@" $.Release.Name "-" .dbchart ":" .port "/" .dbconn | join ""  | b64enc |  quote }}
# {{- ( printf "%s%s:%s@%s-%s%s" "mongodb://" .username .password $.Release.Name "database" ":27017/guestbook?authSource=admin" ) | b64enc | quote }}
{{- end }}
```

adding backend mongodb_uri password and username into global values.yaml file to overwrite the credentials
```yaml
backend:
  secret:
    mongodb_uri:
      username: admin
      password: password
```

check the manifest, upgrade the chart, check the pods and web app
```bash
helm template guestbook | less
helm upgrade demo-guestbook guestbook
kubectl get po
x-www-browser http://frontend.minikube.local
```
## Installing dev and test releases

add if condition into backend ingress.yaml to make it optional whether ingress is enabled so content will be rendered
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ include "backend.fullname" . }}-ingress
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        backend:
          serviceName: {{ include "backend.fullname" . }} 
          servicePort: 80
{{- end }}
```
in guestbook, add a template with ingress and NOTES files in it.
```bash
cd guestbook && mkdir templates && cd templates && touch ingress.yaml && touch NOTES.txt
```
create the ingress.yaml and Notes.txt with vim
ingress.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}-ingress
spec:
  rules:
{{- range .Values.ingress.hosts }}
  - host: {{ $.Release.Name }}.{{ .host.domain }}
    http:
      paths:
      - path: /
        backend:
          serviceName: {{ $.Release.Name}}-{{ .host.chart }}
          servicePort: 80
{{- end }}
```
NOTES.txt
```text
Congratulations ! You installed {{ .Chart.Name }} chart sucessfully.
Release name is {{ .Release.Name }}

You can access the Guestbook application at the following urls :
{{- range .Values.ingress.hosts }}
  http://{{ $.Release.Name }}.{{ .host.domain }}
{{- end }}
Have fun !
```
change the global values.yaml file
```yaml
backend:
  secret:
    mongodb_uri:
      username: admin
      password: password
  ingress:
    enabled: false
frontend: 
  ingress:
    enabled: true

ingress:
  hosts:
    - host:
        domain: frontend.minikube.local
        chart: frontend
    - host:
        domain: backend.minikube.local
        chart: backend
```
add dev and test host ip and hostnames in /etc/hosts file
```bash
{minikube ip} dev.frontend.minikube.local
{minikube ip} dev.backend.minikube.local
{minikube ip} test.frontend.minikube.local
{minikube ip} test.backend.minikube.local
```
check the manifest in the same directory with guestbook if everything is good to go. And delete the latest guestbook release
```bash
helm template guestbook | less
helm list --short
helm uninstall demo-guestboook 
```
install the releases dev and test separately
```bash
helm install dev guestbook --set frontend.config.guestbook_name=DEV
helm install test guestbook --set frontend.config.guestbook_name=TEST
```
## Packaging the chart guestbook and uploading them to a HTTP server for later usage 
packaging frontend backend and database at first and move them in dist directory
```bash
cd ../guestbook 
mv guestbook/charts dist && cd dist
helm package frontend backend database
```
generate an index.yaml for these archieve files
```bash
helm repo index .
```
installing ChartMuseum server 
get the right release from [here](https://github.com/helm/chartmuseum/releases)
for macos
```bash
wget https://get.helm.sh/chartmuseum-v0.13.1-darwin-amd64.tar.gz
tar -xzf chartmuseum-v0.13.1-darwin-amd64.tar.gz && cd darwin-amd64
chmod u+x chartmuseum && sudo mv chartmuseum /usr/local/bin
mkdir -p ~/helm/repo
chartmuseum --storage="local" --storage-local-rootdir=/Users/serdar/helm/repo
```
copy the chart archives from dist dirctory to ~/helm/repo directory
```bash
rm -rf chartmuseum-v0.13.1-darwin-amd64.tar.gz
cp *.tgz ~/helm/repo
```
get the list of charts in the server by making HTTP request
```bash
curl http://localhost:8080/api/charts | jq .
```
## Managing dependecy
adding chartmuseum repo
```bash
helm repo add chartmuseum http://localhost:8080
helm repo list
helm repo update
helm search repo chartmuseum
```
adding dependency in Chart.yaml in guestbook and define the dependencies
```yaml
dependencies:
  - name: backend
    version: ~1.2.2
    repository: http://localhost:8080
  - name: frontend
    version: ^1.2.0
    repository: http://localhost:8080
  - name: database
    version: ~1.2.2
    repository: http://localhost:8080
```
updating dependency for the chart, so dependency will be created for chart
```bash
helm dependency update guestbook
helm dependency list guestbook
```
installing dev release of guestbook with the new dependency archieves
```bash
helm install dev guestbook
```
at this point, you are going to see Chart.lock file in guestbook
when update is needed
```bash
helm dependency build guestbook
```

## Control dependency with conditions and tags
for this part,we are going to install guestbook without backend and database
adding condition and tag to the Chart.yaml file in guestbook 
```yaml
dependencies:
  - name: backend
    version: ~1.2.2
    repository: http://localhost:8080
    condition: backend.enabled
    tags:
      - api
  - name: frontend
    version: ^1.2.0
    repository: http://localhost:8080
  - name: database
    version: ~1.2.2
    repository: http://localhost:8080
    condition: database.enabled
    tags:
      - api
```
edit values.yaml file in guestbook
```yaml
backend:
  enabled: true
  secret:
    mongodb_uri:
      username: admin
      password: password
  ingress:
    enabled: false

frontend: 
  ingress:
    enabled: true

database:
  enabled: true
tags:
 api: true
 
ingress:
  hosts:
    - host:
        domain: frontend.minikube.local
        chart: frontend
    - host:
        domain: backend.minikube.local
        chart: backend
```
install the chart without database and backend through condition set
```bash
helm uninstall dev
helm install dev guestbook --set backend.enabled=false --set database.enabled=false
```
so, you will see only dev-frontend pod
install the chart without database and backend through tag set
```bash
helm uninstall dev
helm install dev guestbook --set tags.api=false
```
## Create guestbook webapp by using existing bitnami/mongodb chart

remove database archive file from guestbook/charts directory and search for mongodb chart 
```bash
rm database-1.2.2.tgz
helm search repo mongodb
```
check the mongodb in artifacthub.io and look for bitnami/mongodb version 7.8.6
change the Chart.yaml in guestbook
```yaml
apiVersion: v2
name: guestbook
appVersion: "2.0"
description: A Helm chart for Guestbook 2.0 
version: 1.2.2
type: application

dependencies:
  - name: backend
    version: ~1.2.2
    repository: http://localhost:8080
    condition: backend.enabled
    tags:
      - api
  - name: frontend
    version: ^1.2.0
    repository: http://localhost:8080
  - name: mongodb
    version: 7.8.x
    repository: https://charts.bitnami.com/bitnami
    condition: mongodb.enabled
    tags:
      - api
```
update helm dependency
```bash
helm dependency update guestbook
helm dependency list guestbook
```
change value.yaml in guestbook: change database into mongodb and mongobd_uri in backend.secret part 
```yaml
backend:
  enabled: true
  secret:
    mongodb_uri:
      username: root
      password: password
      dbchart: mongodb
      dbconn: "guestbook?authSource=admin&replicaSet=rs0"
  ingress:
    enabled: false
frontend:
  ingress:
    enabled: false

mongodb:
  replicaSet:
    enabled: true
    key: password
  mongodbRootPassword: password
  persistence:
    size: 100Mi

tags:
  api: true

ingress:
  hosts:
    - host:
        domain: frontend.minikube.local
        chart: frontend
    - host:
        domain: backend.minikube.local
        chart: backend
```
install the new dev release of guestbook chart 
```bash
helm uninstall dev
helm install dev guestbook
```
now you can browse the webapp via http://dev.frontend.minikube.local or http://test.frontend.minikube.local

## Installing Wordpress with Helm

```bash
helm search repo wordpress
helm install demo-wordpress bitnami/wordpress
kubectl get pods
helm list --short
```
start using wordpress by following the instructions found in Notes after installation of the wordpress chart











