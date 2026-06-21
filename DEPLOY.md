# Guia de Implantação — api-produto (Kubernetes + ArgoCD + GitHub Actions)

Runbook com **todos os comandos** para implantar a aplicação no Kubernetes usando
GitOps (ArgoCD) e CI (GitHub Actions).

> **Convenções usadas abaixo** (ajuste para o seu ambiente):
> - `DOCKERHUB_USER=rsprazeres`
> - Registry: **Docker Hub** → imagem `rsprazeres/api-produto`
> - `REPO=api-produto`
> - `NAMESPACE=api-produto`
> - **Cluster Kubernetes e ArgoCD já estão instalados** (não cobertos aqui)

---

## 0. Pré-requisitos / Validação das ferramentas

Cluster e ArgoCD já provisionados. Validar apenas o que falta na máquina local:

```bash
# kubectl (já presente — v1.35.x) — traz o kustomize EMBUTIDO
kubectl version --client
kubectl kustomize --help >/dev/null && echo "kubectl kustomize OK (use 'kubectl apply -k')"

# kustomize standalone NÃO é necessário localmente (usamos o embutido no kubectl).
# Caso queira o binário standalone mesmo assim:
#   curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
#   sudo mv kustomize /usr/local/bin/

# argocd CLI (opcional — dá para operar tudo via 'kubectl apply'):
#   curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
#   sudo install -m 555 argocd /usr/local/bin/argocd && rm argocd

# GitHub CLI (para configurar secrets do repositório)
type gh || (sudo apt update && sudo apt install gh -y)
gh auth login
```

---

## 1. SEGURANÇA — remover credenciais versionadas (FAZER PRIMEIRO)

```bash
cd /home/rogerio_prazeres/workspace/pessoal/api-produto

# Remover kubeconfig com credenciais reais do cluster do versionamento
git rm --cached kubeconfig.yml

# Ignorar arquivos sensíveis dali em diante
printf '%s\n' 'kubeconfig.yml' '*.kubeconfig' '.env' 'node_modules/' >> .gitignore

git add .gitignore
git commit -m "security: remove kubeconfig versionado e adiciona .gitignore"
```

> ⚠️ A credencial já exposta no histórico do git deve ser **rotacionada** no provedor
> do cluster (ex.: AKS), pois continua acessível em commits antigos.

---

## 2. Build e push da imagem no Docker Hub (validação local — opcional)

```bash
cd /home/rogerio_prazeres/workspace/pessoal/api-produto

# Login no Docker Hub
docker login -u rsprazeres

# Build + tag pelo SHA do commit
set TAG $(git rev-parse --short HEAD)
docker build -t rsprazeres/api-produto:$TAG -f ./src/Dockerfile ./src

# Push (tag do SHA + latest)
docker push rsprazeres/api-produto:$TAG
docker tag  rsprazeres/api-produto:$TAG rsprazeres/api-produto:latest
docker push rsprazeres/api-produto:latest
```

---

## 3. Configurar secrets do repositório (GitHub Actions)

O workflow `.github/workflows/ci.yaml` faz login no Docker Hub e o write-back da tag.

```bash
cd /home/rogerio_prazeres/workspace/pessoal/api-produto

# Credenciais do Docker Hub (use um Access Token, não a senha da conta)
gh secret set DOCKERHUB_USERNAME --body "rsprazeres"
gh secret set DOCKERHUB_TOKEN    --body "<access_token_do_dockerhub>"

# (Opcional) PAT com permissão de push para o write-back da tag no Git.
# Se omitido, o workflow usa o GITHUB_TOKEN nativo.
gh secret set GITOPS_TOKEN --body "<PAT_com_permissao_de_push>"

# Conferir
gh secret list
```

> Gere o Access Token em: Docker Hub → Account Settings → Security → New Access Token.

---

## 4. ArgoCD (já instalado) — acesso e login

```bash
# Senha inicial do admin (se ainda não trocada)
argocd admin initial-password -n argocd

# Acessar a UI (port-forward) — em outro terminal
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login via CLI (opcional)
argocd login localhost:8080 --username admin \
  --password "<senha>" --insecure
```

---

## 5. Criar o namespace da aplicação e o Secret do MongoDB

```bash
# Namespace da aplicação (também é criado pela Application via CreateNamespace=true)
kubectl create namespace api-produto

# Secret com as credenciais do MongoDB (consumido pelos manifestos)
kubectl create secret generic mongodb-credentials \
  --namespace api-produto \
  --from-literal=MONGO_INITDB_ROOT_USERNAME='mongouser' \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD='<senha_forte>' \
  --from-literal=MONGODB_URI='mongodb://mongouser:<senha_forte>@mongodb-service:27017/admin'
```

> Para GitOps "puro", o ideal é usar **Sealed Secrets** ou **External Secrets**
> para versionar o segredo de forma cifrada (ver passo 8).

---

## 6. Validar os manifestos Kustomize localmente (antes de aplicar)

```bash
cd /home/rogerio_prazeres/workspace/pessoal/api-produto

# Renderizar o overlay de produção usando o kustomize embutido no kubectl
kubectl kustomize k8s/overlays/prod

# (Opcional) aplicar manualmente para um teste rápido, sem ArgoCD
# kubectl apply -k k8s/overlays/prod
```

---

## 7. Registrar o repositório e criar a Application do ArgoCD

A forma declarativa (recomendada) usa o manifesto `k8s/argocd/application.yaml`:

```bash
# Aplicar a Application (faz sync automático para k8s/overlays/prod)
kubectl apply -n argocd -f k8s/argocd/application.yaml

# Acompanhar
kubectl get applications -n argocd
kubectl describe application api-produto -n argocd
```

Alternativa via CLI do ArgoCD:

```bash
# Repositório privado (com token); para público, omita --username/--password
argocd repo add https://github.com/rogerio-tecinfo/api-produto.git \
  --username rsprazeres --password "<PAT>"

argocd app create api-produto \
  --repo https://github.com/rogerio-tecinfo/api-produto.git \
  --path k8s/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace api-produto \
  --sync-policy automated --auto-prune --self-heal

argocd app sync api-produto
argocd app wait api-produto --health
```

---

## 8. (Recomendado) Sealed Secrets para versionar segredos com segurança

```bash
# Instalar o controller no cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Instalar o CLI kubeseal
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep tag_name | cut -d '"' -f4 | tr -d 'v')
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Gerar o SealedSecret a partir de um Secret (cifrado, pode ir para o Git)
kubectl create secret generic mongodb-credentials \
  --namespace api-produto \
  --from-literal=MONGODB_URI='mongodb://mongouser:<senha>@mongodb-service:27017/admin' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > k8s/base/mongodb-sealedsecret.yaml

# Lembrar de adicionar 'mongodb-sealedsecret.yaml' ao resources do k8s/base/kustomization.yaml
git add k8s/base/mongodb-sealedsecret.yaml k8s/base/kustomization.yaml
git commit -m "feat: adiciona SealedSecret do MongoDB"
git push
```

---

## 9. Verificação e troubleshooting

```bash
# Estado da Application no ArgoCD
kubectl get application api-produto -n argocd
argocd app get api-produto          # se a CLI estiver instalada

# Recursos no cluster
kubectl get all -n api-produto
kubectl get pods -n api-produto -w

# Logs da API
kubectl logs -n api-produto deploy/api -f

# Testar a aplicação (port-forward)
kubectl port-forward -n api-produto svc/api-service 8080:80
curl http://localhost:8080/health
curl http://localhost:8080/api/produto
```

---

## 10. Fluxo contínuo (após configurado)

```bash
# 1. Desenvolvedor faz alteração e push
git add .
git commit -m "feat: nova funcionalidade"
git push

# 2. GitHub Actions builda e publica a imagem no Docker Hub (tag = SHA)
#    e atualiza a tag no overlay de produção (write-back automático)

# 3. ArgoCD detecta a mudança no Git e sincroniza o cluster automaticamente
#    (syncPolicy automated). Acompanhar:
kubectl get application api-produto -n argocd
```

---

## 11. Rollback (se necessário)

```bash
# Via CLI do ArgoCD
argocd app history api-produto
argocd app rollback api-produto <ID_DA_REVISAO>

# Ou via Git (reverter o commit da tag e deixar o ArgoCD re-sincronizar)
git revert <sha_do_commit>
git push
```

---

### Estrutura de arquivos criada

```
.github/workflows/ci.yaml          # Pipeline: build + push (Docker Hub) + write-back da tag
k8s/base/                          # Manifestos base (Kustomize)
  ├── namespace.yaml
  ├── mongodb.yaml                 # StatefulSet + Service (com PVC)
  ├── api-deployment.yaml          # Deployment com probes + resources + Secret
  ├── api-service.yaml
  └── kustomization.yaml
k8s/overlays/dev/kustomization.yaml   # 1 réplica
k8s/overlays/prod/kustomization.yaml  # 2 réplicas (tag atualizada pelo CI)
k8s/argocd/application.yaml        # Application declarativa do ArgoCD
src/.dockerignore                  # Otimiza o build da imagem
```

> O antigo `k8s/deployment.yaml` (monólito com senha em texto puro) foi **substituído**
> por esta estrutura — pode ser removido após validar a nova implantação.
