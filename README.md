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

There are more charts. check this helm hub out
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
```
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

```bahs
helm template [chart name] or
helm install [release name] [chart name] --dry-run --debug
```
Customizing frontend chart
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

the same changes apply for backend and database charts

