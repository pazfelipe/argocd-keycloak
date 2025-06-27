# Setup Local Argo CD + Keycloak OIDC Integration

Este README descreve os passos para implantar e configurar o Keycloak e o Argo CD em um cluster local k3d usando Helm, incluindo ajustes de host e port-forward.

## Prerequisitos

- k3d, kubectl e Helm instalados
- Acesso com sudo para port-forward em portas privilegiadas (80/443)

## Estrutura do repositório

```bash
.
└── helm
    ├── argocd
    │   └── values.yaml
    └── keycloak
        ├── ingress.yaml
        └── values.yaml
```
## Como executar

### 1. Criar o cluster k3d

```bash
# Remover cluster existente
k3d cluster delete k3d-dev`
```
```bash
# Criar novo cluster
k3d cluster create k3d-dev
```
### 2. Instalar Ingress NGINX

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### 3. Configurar /etc/hosts

```bash
echo "127.0.0.1 argocd.local keycloak.local" | sudo tee -a /etc/hosts
```

### 4. Implantação do Keycloak via Helm

```bash
# Adicionar repositório Bitnami e atualizar
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```
```bash
# Instalar Keycloak com values.yaml em helm/keycloak/values.yaml
helm install keycloak bitnami/keycloak \
  --namespace keycloak --create-namespace \
  -f helm/keycloak/values.yaml
```

### 4.1. Ingress do Keycloak

```bash
# Aplicar ingress helm/keycloak/ingress.yaml
kubectl apply -f helm/keycloak/ingress.yaml
```

### 5. Implantação do Argo CD via Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

```bash
# Instalar Argo CD com values.yaml em helm/argocd/values.yaml
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  -f helm/argocd/values.yaml
```

### 6. Patching do Argo CD Server

Informe o IP interno do Ingress (`<ING_IP>`) e aplique:

```bash
ING_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.clusterIP}')
echo $ING_IP
```

```
kubectl patch deployment argocd-server -n argocd --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/hostAliases",
    "value": [
      { "ip": "<ING_IP>", "hostnames": ["keycloak.local"] }
    ]
  }
]'
```
### 7. Port-forwarding

```bash
# Ingress NGINX → porta 80 do host (acesso sem especificar porta)
sudo kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 80:80
```
```bash
# Argo CD → porta 443 do host acessível em 8081
git kubectl port-forward svc/argocd-server -n argocd 8081:443
```

### 8. Acessos
	- Keycloak: http://keycloak.local/admin/
	- Argo CD: https://argocd.local:8081

Observação: substitua `<ING_IP>` pelo IP do service `ingress-nginx-controller` obtido com:

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.clusterIP}'
```

## Configuração do Client no Keycloak

Acesse o console do Keycloak em http://keycloak.local/admin/ e faça login.

> Se for o primeiro acesso, o username é `admin` e a senha é `changeme`

> Você pode recuperar a senha inicial rodando o comando

```bash
kubectl get secret --namespace keycloak keycloak -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

No menu lateral esquerdo, clique em `Clients`.

Clique no botão `Create client`.

Em `Client ID`, digite **argocd** e clique em `Next`.

Ative `Client authentication` e garanta que `Standard Flow` esteja marcado, desmarcando os demais flows. Clique em `Next`.

Na tela de configurações:

- Root URL: https://argocd.local:8081
- Home URL: https://argocd.local:8081
- Valid Redirect URIs: https://argocd.local:8081/*
- Valid Post Logout Redirect URIs: https://argocd.local:8081/
- Admin URL: https://argocd.local:8081/
- Web Origins: https://argocd.local:8081

Em seguida, clique em `Save`.

Após criar o client, vá até a aba `Credentials`, copie o valor de Secret e atualize o campo clientSecret em [helm/argocd/values.yaml](helm/argocd/values.yaml).

```yaml
clientSecret: <valor_do_secret_do_client>
```

Reaplique o chart e reinicie o servidor do Argo CD:

```bash
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  -f helm/argocd/values.yaml
kubectl rollout restart deployment argocd-server -n argocd
```
Pronto! O client argocd está configurado no Keycloak para autenticação OIDC com o Argo CD.


## Extra

10. Forçar login apenas via Keycloak

Edite o arquivo [helm/argocd/values.yaml](helm/argocd/values.yaml) e atualize para:

```yaml
dex:
  enabled: false

configs:
  cm:
    admin.enabled: "false"
    url: https://argocd.local:8081
    oidc.config: |
      name: Keycloak
      issuer: http://keycloak.local/realms/master
      clientID: argocd
      clientSecret: <valor_do_secret_do_client>
      requestedScopes: ["openid", "profile", "email"]
      insecure: true
```

Note o novo campo `admin.enabled`

Salve e aplique:

```bash
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  -f helm/argocd/values.yaml
kubectl rollout restart deployment argocd-server -n argocd
```