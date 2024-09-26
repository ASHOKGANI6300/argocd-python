# Flask Python Projesi ve ArgoCD ile Kubernetes Dağıtımı deneme
Bu makalede, basit bir Flask uygulamasını Docker ile paketleyip, Docker Hub'a gönderme ve ArgoCD ile Kubernetes cluster'a otomatik dağıtım yapma sürecini adım adım inceleyeceğiz. Bu süreçte GitHub Actions kullanarak CI/CD pipeline'ı oluşturacağız. GitHub Actions aracılığıyla uygulama kodları her güncellendiğinde Docker imajı otomatik olarak oluşturulacak ve Kubernetes’e dağıtılacaktır. 

# Proje Yapısı
Dockerfile: Flask uygulaması için Docker imajını oluşturur.

app.py: Flask uygulamasının kaynak kodu.

requirement.txt: Uygulamanın bağımlılıklarını içerir.

templates/index.html: Flask tarafından kullanılan HTML şablon dosyası.

manifest/deployment.yaml: Kubernetes deployment manifest dosyası.
.github/workflows/docker-build.yml: GitHub Actions workflow komut dosyası.

# Gereksinimler
Docker Hub hesabı

GitHub repository

ArgoCD kurulumu

Kubernetes Cluster

kubectl komut satırı aracı

# 1. Dockerfile ile Flask Uygulaması
Flask uygulamasını Docker imajına dönüştürmek için aşağıdaki Dockerfile'ı kullanabilirsiniz:

Dockerfile
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
requirement.txt dosyası Flask uygulamasının ihtiyaç duyduğu kütüphaneleri içerir:
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
app.py dosyası, basit bir Flask uygulamasını içerir:
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
HTML şablon dosyası templates/index.html:
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
# 2. Kubernetes Deployment Manifesti
Kubernetes manifest dosyası olan manifest/deployment.yaml:
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaskapp
  labels:
    app: flask 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask 
  template:
    metadata:
      labels:
        app: flask 
    spec:
      containers:
      - name: flaskapp
        image: username/argocd-python:v1.0
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: flaskservice
spec:
  selector:
    app: flask
  ports:
    - port: 80
      targetPort: 5000
  type: LoadBalancer
```
# 3. GitHub Actions Workflow
CI/CD pipeline'ımızı oluşturan GitHub Actions workflow dosyası .github/workflows/docker-build.yml:

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
# 4. ArgoCD Kurulumu ve Manifest İzleme
ArgoCD'yi kurmak ve GitHub repository'deki manifest dosyalarını izlemek için şu adımları izleyebilirsiniz:

ArgoCD kurulumu:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.1.3/manifests/install.yaml
```
ArgoCD ile uygulamanın dağıtımı için application.yaml dosyasını oluşturun:
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
ArgoCD üzerinde bu uygulamayı başlatmak için:
```bash
kubectl apply -f application.yaml
```
ArgoCD’ye erişim için LoadBalancer hizmetini başlatın:
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
  - port: 443
    targetPort: 8080
```
```bash
kubectl apply -f argocd-service.yaml
kubectl get svc -n argocd
```
ArgoCD'ye erişim sağladıktan sonra, uygulamanızın dağıtımını ve güncellemelerini otomatik olarak yönetebilirsiniz.
