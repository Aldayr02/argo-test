## 1. Instalación de K3s

### Paso 1: Instalar K3s

```bash
# Instalación básica de K3s
curl -sfL https://get.k3s.io | sh -

```

### Paso 2: Verificar la instalación

```bash
# Verificar que K3s está corriendo
sudo systemctl status k3s

# Verificar los nodos
sudo k3s kubectl get nodes

# Configurar kubectl para usuario no-root (opcional)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
```

## 2. Instalación de ArgoCD

### Paso 1: Crear namespace y instalar ArgoCD

```bash
# Crear namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Paso 2: Esperar a que los pods estén listos

```bash
# Verificar que todos los pods están corriendo
kubectl get pods -n argocd

# Esperar a que estén todos en Running
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Paso 3: Exponer ArgoCD UI

```bash
# Opción 1: Port-forward (para desarrollo)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Opción 2: Cambiar a NodePort (para acceso externo)
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
```

### Paso 4: Obtener contraseña inicial

```bash
# Obtener la contraseña inicial del admin
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
# Instalar Helm si no lo tienes
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Agregar el repositorio de Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Crear namespace para Crossplane
kubectl create namespace crossplane-system

# Instalar Crossplane
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
