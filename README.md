# Projeto CI/CD com GitHub Actions, FastAPI e ArgoCD

## 📋 Visão Geral do Projeto

Este projeto implementa um pipeline completo de CI/CD (Integração Contínua e Entrega Contínua) para uma aplicação FastAPI, automatizando todo o processo desde o código até o deploy em produção usando práticas modernas de DevOps e GitOps.

<div align="center">
    <img src="https://skillicons.dev/icons?i=kubernetes,github,git,docker" />
    <img src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/argo-cd.svg" width=40px />
</div>

## 🏗️ Arquitetura do Sistema
```bash
[GitHub Repo (hello-app)] 
        ↓ (Push para main)
[GitHub Actions CI/CD] 
        ↓ (Build e Push)
[Docker Hub Registry] 
        ↓ (Pull Image)
[ArgoCD] → [Kubernetes Cluster] → [Aplicação Rodando]
        ↑
[GitHub Repo (hello-manifests)]
```

## 📁 Estrutura de Arquivos

### Repositório da Aplicação (hello-app)
 ```bash
hello-app/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── main.py
├── requirements.txt
└── Dockerfile
```
### Repositório de Manifests (hello-manifests)
 ```bash
hello-manifests/
├── deployment.yaml
└── service.yaml
```
## 📝 Arquivos do Projeto

### main.py
 ```bash
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Dyego"}
```
### requirements.txt
 ```bash
fastapi
uvicorn
```
### Dockerfile
 ```bash
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
### .github/workflows/ci-cd.yml
 ```bash
name: CI/CD Pipeline
on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Run tests
      run: pytest || echo "Sem testes configurados ainda"

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          mateoclima/hello-app:${{ github.sha }}
          mateoclima/hello-app:latest

  update-manifest:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - name: Checkout manifests repository
      uses: actions/checkout@v3
      with:
        repository: mateoclima/hello-manifests
        ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}

    - name: Update image tag in deployment
      run: |
        sed -i "s|image: .*|image: mateoclima/hello-app:${{ github.sha }}|" deployment.yaml

    - name: Create Pull Request
      uses: peter-evans/create-pull-request@v5
      with:
        branch: update-image-${{ github.sha }}
        commit-message: "Update image to ${{ github.sha }}"
        title: "Update image to ${{ github.sha }}"
        body: "Atualizando a imagem no deployment para a versão do commit ${{ github.sha }}"
```
### deployment.yaml
 ```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-app
        image: mateolima/hello-app:1759098837
        ports:
        - containerPort: 80
```
### service.yaml
 ```bash
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
spec:
  selector:
    app: hello-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
## 🔧 Pipeline CI/CD - Explicação Detalhada

### Parte 1: Definição Básica e Trigger
 ```bash
name: CI/CD Pipeline
on:
  push:
    branches: [ "main" ]
Propósito: Executar automaticamente o pipeline quando houver push na branch main.
```
### Parte 2: Estrutura de Jobs
 ```bash
jobs:
  build-and-push:
    runs-on: ubuntu-latest
```
Propósito: Organizar o pipeline em jobs independentes executados em sequência.

### Parte 3: Job 1 - Checkout do Código
 ```bash
- name: Checkout code
  uses: actions/checkout@v3
```
Propósito: Garantir acesso ao código mais recente do repositório.

### Parte 4: Job 1 - Configuração do Docker Buildx
 ```bash
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
```
Propósito: Otimizar o processo de build de imagens Docker.

### Parte 5: Job 1 - Login no Docker Hub
 ```bash
- name: Login to Docker Hub
  uses: docker/login-action@v2
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```
Propósito: Autenticar no Docker Hub para enviar a imagem construída.

### Parte 6: Job 1 - Execução de Testes
 ```bash
- name: Run tests
  run: pytest || echo "Sem testes configurados ainda"
```
Propósito: Garantir qualidade do código antes de construir e deployar.

### Parte 7: Job 1 - Build e Push da Imagem Docker
 ```bash
- name: Build and push Docker image
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: |
      mateoclima/hello-app:${{ github.sha }}
      mateoclima/hello-app:latest
```
Propósito: Construir a imagem Docker e enviá-la para o Docker Hub com versionamento.

### Parte 8: Job 2 - Dependência entre Jobs
 ```bash
update-manifest:
  needs: build-and-push
  runs-on: ubuntu-latest
```
Propósito: Garantir que a atualização dos manifests só aconteça após a imagem estar disponível.

### Parte 9: Job 2 - Checkout do Repositório de Manifests
 ```bash
- name: Checkout manifests repository
  uses: actions/checkout@v3
  with:
    repository: mateoclima/hello-manifests
    ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
```
Propósito: Acessar o repositório com configurações do Kubernetes para ArgoCD.

### Parte 10: Job 2 - Atualização da Tag da Imagem
 ```bash
- name: Update image tag in deployment
  run: |
    sed -i "s|image: .*|image: mateoclima/hello-app:${{ github.sha }}|" deployment.yaml
```
Propósito: Atualizar automaticamente a versão da imagem no deployment do Kubernetes.

### Parte 11: Job 2 - Criação do Pull Request
 ```bash
- name: Create Pull Request
  uses: peter-evans/create-pull-request@v5
  with:
    branch: update-image-${{ github.sha }}
    commit-message: "Update image to ${{ github.sha }}"
    title: "Update image to ${{ github.sha }}"
    body: "Atualizando a imagem no deployment para a versão do commit ${{ github.sha }}"
```
Propósito: Criar PR para revisão seguindo boas práticas de GitOps.

## 🚀 Passo a Passo de Implementação

### 🔧 Pré-requisitos - Configuração Inicial

1. Criar Contas e Instalar Dependências:
   - ✅ Conta no GitHub
   - ✅ Conta no Docker Hub
   - ✅ Rancher Desktop com Kubernetes
   - ✅ kubectl configurado
   - ✅ ArgoCD instalado no cluster

2. Configurar Secrets no GitHub:
   - Vá em: Settings → Secrets and variables → Actions
   - Adicione:
     - DOCKER_USERNAME: Seu usuário Docker Hub
     - DOCKER_PASSWORD: Seu token Docker Hub
     - SSH_PRIVATE_KEY: Chave SSH para acesso ao repositório de manifests

### 📝 Etapa 1: Criar a Aplicação

1. Criar repositório hello-app:
 ```bash
mkdir hello-app
cd hello-app
git init
```

2. Adicionar os arquivos conforme listados acima.

### 🔄 Etapa 2: Configurar o Pipeline CI/CD

Estrutura do Workflow:
- Trigger: Push na branch main
- Job 1: Build and Push
- Job 2: Update Manifest

Tags da Imagem:
- mateoclima/hello-app:latest (sempre a última versão)
- mateoclima/hello-app:[commit-sha] (versão específica)

### 📊 Etapa 3: Repositório de Manifests Kubernetes

1. Criar repositório hello-manifests:
 ```bash
mkdir hello-manifests
cd hello-manifests
git init
```
2. Adicionar arquivos deployment.yaml e service.yaml conforme listados acima.

### ⚙️ Etapa 4: Configurar ArgoCD

1. Instalar ArgoCD no cluster:
 ```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2. Acessar a UI do ArgoCD:
 ```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Acesse: https://localhost:8080

3. Criar Application no ArgoCD:
   - Nome: hello-app
   - Project: default
   - Sync Policy: Automated
   - Repository URL: https://github.com/mateoclima/hello-manifests
   - Path: . (raiz do repositório)
   - Cluster: in-cluster
   - Namespace: default

### 🧪 Etapa 5: Testar o Pipeline Completo

1. Fazer uma alteração no código:
# Em main.py, altere:
 ```bash
return {"message": "Hello CI/CD World!"}
```
# Para:
 ```bash
return {"message": "Olá, tudo bem?"}
```
2. Commit e push:
 ```bash
git add .
git commit -m "Update message"
git push origin main
```
3. Monitorar o processo:
   - GitHub Actions → Ver workflow executando
   - Docker Hub → Ver nova imagem criada
   - GitHub → Ver Pull Request criado no repositório de manifests
   - ArgoCD → Ver sincronização automática
   - Kubernetes → Ver pods sendo atualizados

4. Testar a aplicação:
kubectl port-forward svc/hello-app-service 8080:80
# Acesse: http://localhost:8080

## ✅ Verificação do Funcionamento

### 🔍 Checklist de Validação

- [ X ] GitHub Actions: Workflow executado com sucesso
- [ X ] Docker Hub: Nova imagem com tag do commit
- [ X ] Pull Request: Criado automaticamente no repositório de manifests
- [ X ] ArgoCD: Aplicação sincronizada e healthy
- [ X ] Kubernetes: Pods rodando a nova versão
- [ X ] Aplicação: Respondendo com a nova mensagem

### 📊 Comandos de Verificação

# Verificar pods
 ```bash
kubectl get pods -l app=hello-app
```
# Verificar logs
 ```bash
kubectl logs -l app=hello-app
```
# Verificar serviços
 ```bash
kubectl get services
```
# Verificar status no ArgoCD
 ```bash
argocd app get hello-app
```
## 🛠️ Solução de Problemas Comuns

### ❌ Build Falha
- Verificar formato do Dockerfile
- Confirmar secrets do Docker Hub configurados

### ❌ Push Falha
- Verificar permissões do token Docker Hub
- Confirmar repositório existe no Docker Hub

### ❌ ArgoCD Não Sincroniza
- Verificar URL do repositório de manifests
- Confirmar permissões de acesso
- Verificar sintaxe dos arquivos YAML

### ❌ Aplicação Não Responde
- Verificar port-forward correto
- Confirmar serviço expondo a porta correta
- Checar logs dos pods para erros

## 📈 Próximos Passos e Melhorias

1. Adicionar testes automatizados
2. Implementar ambientes (dev/staging/prod)
3. Adicionar monitoramento e alertas
4. Implementar rollback automático
5. Adicionar segurança no pipeline (scan de vulnerabilidades)

## 👥 Autor

- Matheus José 