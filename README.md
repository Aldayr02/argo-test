## 1. Instalación de K3s

### Paso 1: Instalar K3s

```bash
curl -sfL https://get.k3s.io | sh -

```

### Paso 2: Verificar la instalación

```bash

sudo systemctl status k3s

sudo k3s kubectl get nodes

mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

sudo yum install bash-completion
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

source ~/.bashrc

```

## 2. Instalación de ArgoCD

### Paso 1: Crear namespace y instalar ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Paso 2: Esperar a que los pods estén listos

```bash
kubectl get pods -n argocd

kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Paso 3: Exponer ArgoCD UI

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
```

### Paso 4: Obtener contraseña inicial

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Usuario: admin
# Contraseña: (la obtenida del comando anterior)
```

### Paso 5: Instalar ArgoCD CLI (opcional)

```bash
# Descargar ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login via CLI
argocd login localhost:8080 --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

## 3. Instalación de Crossplane

### Paso 1: Instalar Crossplane usando Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

kubectl create namespace crossplane-system

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace
```

### Paso 2: Verificar la instalación de Crossplane

```bash
# Verificar que los pods están corriendo
kubectl get pods -n crossplane-system

# Verificar que los CRDs están instalados
kubectl get crd | grep crossplane
```

### Paso 3: Instalar Crossplane CLI

```bash
# Descargar Crossplane CLI
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin

# Verificar instalación
kubectl crossplane --help
```

## 4. Configuración inicial de Crossplane

### Paso 1: Instalar un Provider (ejemplo: AWS)

Crea el archivo `provider-aws.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.44.0
```

Aplicar el provider:

```bash
# Aplicar el provider
kubectl apply -f provider-aws.yaml

# Verificar que el provider está instalado
kubectl get providers
```

### Paso 2: Configurar credenciales (ejemplo para AWS)

```bash
# Crear secret con credenciales de AWS
kubectl create secret generic aws-secret -n crossplane-system \
  --from-literal=creds='[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY'
```

Crea el archivo `provider-config.yaml`:

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
```

Aplicar la configuración:

```bash
kubectl apply -f provider-config.yaml
```


## 6. Verificación final

```bash
# Verificar K3s
kubectl get nodes

# Verificar ArgoCD
kubectl get pods -n argocd

# Verificar Crossplane
kubectl get pods -n crossplane-system
kubectl get providers

# Obtener información de acceso a ArgoCD
echo "ArgoCD UI: http://localhost:8080 (si usas port-forward)"
echo "Usuario: admin"
echo "Contraseña: $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -
