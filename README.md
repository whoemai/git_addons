# Gerenciador de Addons K8s via GitOps (Argo CD)

Este repositório foi estruturado utilizando o padrão **App of Apps** do Argo CD combinado com **Helm Wrapper Charts** para gerenciar de forma centralizada e declarativa os addons necessários para qualquer cluster Kubernetes (ex: KEDA, External Secrets, External DNS, etc.).

---

## 🏗️ Arquitetura

O padrão adotado neste repositório resolve de maneira elegante o gerenciamento de dependências e customizações:

1. **Bootstrap (`bootstrap/`)**: Contém o `AppProject` (`project.yaml`) e a `Application` raiz (`root-application.yaml`). Ao aplicar estes manifestos no cluster, o Argo CD cria o escopo de segurança do projeto e passa a monitorar a pasta `apps/`.
2. **Applications (`apps/`)**: Contém manifestos do tipo `Application` do Argo CD. Cada arquivo nesta pasta representa um addon ativado e está associado ao projeto `addons`.
3. **Helm Wrappers (`addons/`)**: Em vez de apontar o Argo CD diretamente para os repositórios Helm públicos, criamos um "Chart wrapper" (envelope) local para cada addon. Isso permite:
   - Declarar e fixar a versão oficial como dependência.
   - Declarar manifestos Kubernetes adicionais diretamente em `addons/<nome-do-addon>/templates/` (ex: `ClusterStore`, `ScaledObjects`, ingressos customizados, etc.).
   - Utilizar uma hierarquia clara de `values.yaml` para customizar as configurações dos subcharts.

---

## 📁 Estrutura de Pastas

```text
git_addons/
├── README.md                      # Este manual de instruções
├── bootstrap/
│   ├── project.yaml               # Argo CD AppProject de infraestrutura (addons)
│   └── root-application.yaml      # Argo CD Application raiz (App of Apps)
├── apps/                          # Argo CD Applications (uma para cada addon)
│   ├── keda.yaml
│   ├── external-secrets.yaml
│   └── external-dns.yaml
└── addons/                        # Wrapper Helm Charts para os addons
    ├── keda/
    │   ├── Chart.yaml             # Define KEDA upstream como dependência
    │   └── values.yaml            # Configurações padrão do KEDA
    ├── external-secrets/
    │   ├── Chart.yaml             # Define External Secrets upstream como dependência
    │   └── values.yaml            # Configurações padrão do External Secrets
    └── external-dns/
        ├── Chart.yaml             # Define External DNS upstream como dependência
        └── values.yaml            # Configurações padrão do External DNS
```

---

## 🚀 Como Inicializar no Cluster (Bootstrap)

### 1. Pré-requisitos
* Um cluster Kubernetes ativo.
* Argo CD instalado no namespace `argocd`. Se não tiver instalado, rode:
  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```

### 2. Atualizar URLs do Repositório Git
Antes de aplicar os manifestos, você precisa apontar as URLs para o seu próprio repositório Git onde esta estrutura foi hospedada.

Faça uma substituição global no projeto da URL exemplo:
`https://github.com/your-username/git_addons.git`
pela **URL do seu repositório Git**.

Arquivos que precisam ser atualizados:
* `bootstrap/project.yaml`
* `bootstrap/root-application.yaml`
* `apps/keda.yaml`
* `apps/external-secrets.yaml`
* `apps/external-dns.yaml`

### 3. Aplicar o AppProject e a Application Raiz
Com o Argo CD rodando no cluster, aplique os manifestos do bootstrap:

```bash
# Aplica primeiro o projeto e depois a aplicação raiz
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/root-application.yaml
```

O Argo CD irá registrar o projeto `addons` e detectar a `Application` raiz (`root-addons`), que por sua vez lerá a pasta `apps/` e começará a provisionar todos os addons (`keda`, `external-secrets` e `external-dns`) de forma isolada e segura.

---

## 🛠️ Como Adicionar um Novo Addon

Adicionar um novo addon (ex: `ingress-nginx`) é simples e segue um fluxo padronizado:

### Passo 1: Criar o Wrapper Helm
Crie uma nova pasta dentro de `addons/` para o seu addon, contendo um arquivo `Chart.yaml` e um `values.yaml`:

**`addons/ingress-nginx/Chart.yaml`**:
```yaml
apiVersion: v2
name: ingress-nginx-addon
description: Wrapper para o Ingress Nginx Controller
type: application
version: 1.0.0
appVersion: "4.10.1"
dependencies:
  - name: ingress-nginx
    version: 4.10.1
    repository: https://kubernetes.github.io/ingress-nginx
```

**`addons/ingress-nginx/values.yaml`**:
```yaml
# Valores para a dependência 'ingress-nginx'
ingress-nginx:
  controller:
    replicaCount: 2
    service:
      type: LoadBalancer
```

### Passo 2: Criar a Application do Argo CD
Crie o arquivo que diz ao Argo CD para implantar seu wrapper chart:

**`apps/ingress-nginx.yaml`**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: addons
  source:
    repoURL: 'https://github.com/SUA-ORGANIZACAO/SEU-REPO.git'
    targetRevision: HEAD
    path: addons/ingress-nginx
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Passo 3: Commitar e Enviar ao Git
Envie as alterações para a sua branch principal (`main` ou `master`):

```bash
git add .
git commit -m "feat: adiciona addon ingress-nginx"
git push origin main
```

O Argo CD raiz (`root-addons`) identificará o novo arquivo na pasta `apps/` e aplicará automaticamente o `ingress-nginx` no cluster!

---

## 📦 Gerenciamento de Dependências do Helm

Como estamos utilizando dependências do Helm nos wrappers locais, o Argo CD precisa ter acesso aos pacotes de dependência (`.tgz`). O Argo CD gerencia isso nativamente executando `helm dependency build` antes de gerar os templates.

Se você quiser validar ou renderizar os charts localmente em sua máquina, execute:

```bash
# Atualiza as dependências de um addon específico
helm dependency update addons/keda/

# Renderiza os templates localmente para inspecionar os recursos gerados
helm template addons/keda/
```
