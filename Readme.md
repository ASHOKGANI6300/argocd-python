# Flask Python Projesi ve ArgoCD ile Kubernetes Dağıtımı
Bu proje, Flask uygulamanızı GitHub Actions kullanarak Docker imajına dönüştürüp Docker Hub’a göndermenizi ve ArgoCD ile Kubernetes’e dağıtmanızı sağlar.

# Proje Yapısı

*** Dockerfile *** Flask uygulaması için Docker imajı oluşturur.

*** app.py *** Flask uygulamanızın kaynak kodu.

*** requirement.txt *** Flask uygulamanızın library bağımlılıkları.

*** templates/index.html *** flask tarafından kullanılan index.template.

*** application.yaml *** Kubernetes'e deployment için kullanılan manifest dosyası.

*** manifest/deployment.yaml *** Kubernetes deployment manifest'i.

*** .github/workflows/docker-build.yml *** Github Actions workflow deployment komut seti

# Gereksinimler
Docker Hub hesabı 

GitHub repository'si

ArgoCD kurulumu

Kubernetes Cluster

Kubectl

Github Actions tanımlanacak değişkenler

DOCKERHUB_USERNAME 

DOCKER_HUB_ACCESS_TOKEN


# 1. Dockerfile ile Flask Uygulaması
## Dockerfile
```bash
FROM python:3.9
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
COPY templates/ ./templates/ 
EXPOSE 5000
CMD ["python", "app.py"]

```

requirement.txt
```bash
blinker==1.6.2
click==8.1.6
colorama==0.4.6
Flask==2.3.2
importlib-metadata==6.8.0
itsdangerous==2.1.2
Jinja2==3.1.2
MarkupSafe==2.1.3
Werkzeug==2.3.6
zipp==3.16.2
```
app.py
```bash
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def hello():
    greeting_message = "Merhaba ArgoCD"
    return render_template('index.html', message=greeting_message)

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```
templates/index.html
```bash
<!DOCTYPE html>
<html lang="tr">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
    <title>ArgoCD Uygulaması</title>
</head>

<body>
    <div class="container mt-5">
        <h1 class="text-center">ArgoCD Uygulamasına Hoşgeldiniz!</h1>
        <div class="alert alert-info text-center" role="alert">
            {{ message }}
        </div>
        <footer class="text-center mt-4">
            <p>© 2024 ArgoCD Uygulaması</p>
        </footer>
    </div>
    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.9.2/dist/umd/popper.min.js"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>
</body>

</html>
```
manifest/deployment.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaskapp
  labels:
    app:  flask 
spec:
  template:
    metadata:
      name: flaskapp
      labels:
        app: flask 
    spec:
      containers:
        - name: myfirstpod
          image: username/argocd-python:v1.0
          ports:
            - containerPort: 5000
  replicas: 1
  selector:
    matchLabels:
      app: flask 
---
apiVersion: v1
kind: Service
metadata:
  name: flaskservice
spec:
  selector:
    app: flask
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
```
# 2. GitHub Actions Workflow
GitHub Actions dosyası .github/workflows/docker-build.yml oluşturun:

```bash
name: Build and Deploy to Kubernetes

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2
      with:
        token: ${{ secrets.PAT }} 

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_HUB_USERNAME }}
        password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

    - name: Build and push Docker image
      run: |
        docker build -t ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:${{ github.run_id }} .
        docker push ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:${{ github.run_id }}

    - name: Update Kubernetes manifest
      run: |
        sed -i 's#image: .*#image: ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:${{ github.run_id }}#g' manifest/deployment.yaml
        git config --local user.email "${{ secrets.EMAIL_ADDRESS }}"  
        git config --local user.name "${{ github.actor }}"  
        
        git add manifest/deployment.yaml
        git commit -m "Update image version to workflow #${{ github.run_id }}"
        git push https://x-access-token:${{ secrets.PAT }}@github.com/${{ github.actor }}/argocd-python.git main   

```
# 3. ArgoCD Kurulumu ve Manifest İzleme
ArgoCD’yi kurun ve repository'nizin manifest klasörünü izlemeye ayarlayın:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.1.3/manifests/install.yaml
```
## ArgoCD dashboard'a giriş yaptıktan sonra, repository'nizdeki manifest klasörünü dinleyecek şekilde ayarlayın.

```bash

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flask-app
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  source:
    path: manifest
    repoURL: 'https://github.com/username/argocd-python'
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
## Argocd Application devreye almak için
```bash
kubectl apply -f application.yaml
```

## ArgoCD'ye erişim sağlamak için:
Loadbalancer Servis
#argocd-service.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: argocd-server
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  - port: 443
    targetPort: 8080
    protocol: TCP
    name: https


```
## ArgoCD  Loadbalancer kurulumu için
```bash
kubectl apply -f argocd-service.yaml
```
## Argocd server loadbalancer ip öğrenmek için
```bash
kubectl get svc -n argocd
```
## Argocd default user admin şifre için aşağıdaki komutu çalıştırın
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```
Loadbalancer ip user admin ve secretten aldığınız şifre ile argocd dashboarda browser üzerinde ulaşabilirsiniz.
