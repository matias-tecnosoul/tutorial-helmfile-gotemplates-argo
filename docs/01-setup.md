# 01 - Setup del Entorno

Esta guía te ayudará a preparar tu máquina con todas las herramientas necesarias para trabajar con Helmfile y Go Templates.

## 🎯 Objetivos

Al finalizar este capítulo tendrás:
- ✅ Todas las herramientas instaladas y funcionando
- ✅ Cluster de Kubernetes local (Kind)
- ✅ Direnv configurado para gestionar variables de entorno
- ✅ Repositorio del tutorial clonado y listo

## 📋 Requisitos del Sistema

- **Sistema Operativo**: Linux (preferentemente Arch/Manjaro, compatible con Debian/Ubuntu)
- **RAM**: Mínimo 8GB (recomendado 16GB)
- **CPU**: 4 cores (mínimo 2)
- **Disco**: 20GB libres
- **Conexión a internet**: Para descargar imágenes y dependencias

## 🛠️ Instalación de Herramientas

### 1. Docker

Docker es esencial para Kind (Kubernetes in Docker).

#### Arch/Manjaro
```bash
# Instalar Docker
sudo pacman -S docker docker-compose

# Habilitar y arrancar el servicio
sudo systemctl enable --now docker

# Agregar tu usuario al grupo docker (evita usar sudo)
sudo usermod -aG docker $USER

# Recargar grupos (o cerrar sesión y volver a entrar)
newgrp docker

# Verificar instalación
docker --version
docker run hello-world
```

**Salida esperada:**
```
Docker version 24.0.7, build afdd53b
Hello from Docker!
```

#### Debian/Ubuntu
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

### 2. kubectl

kubectl es la CLI para interactuar con Kubernetes.

#### Arch/Manjaro
```bash
sudo pacman -S kubectl

# Verificar
kubectl version --client
```

#### Debian/Ubuntu
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verificar
kubectl version --client
```

**Salida esperada:**
```
Client Version: v1.28.0
```

### 3. Helm

Helm es el package manager de Kubernetes.

#### Arch/Manjaro
```bash
sudo pacman -S helm

# Verificar
helm version
```

#### Debian/Ubuntu
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verificar
helm version
```

**Salida esperada:**
```
version.BuildInfo{Version:"v3.13.0"}
```

#### Plugins de Helm

Instalar plugins necesarios:

```bash
# helm-diff: Ver diferencias antes de aplicar
helm plugin install https://github.com/databus23/helm-diff

# helm-secrets: Soporte para secrets (aunque no usemos SOPS aún)
helm plugin install https://github.com/jkroepke/helm-secrets

# Verificar plugins instalados
helm plugin list
```

**Salida esperada:**
```
NAME    VERSION DESCRIPTION
diff    3.9.0   Preview helm upgrade changes as a diff
secrets 4.5.1   Manage secrets with Git workflow
```

### 4. Helmfile

Helmfile es el orquestador de múltiples Helm charts.

#### Instalación desde binario (Recomendado)

```bash
# Descargar última versión
HELMFILE_VERSION="0.159.0"
curl -LO "https://github.com/helmfile/helmfile/releases/download/v${HELMFILE_VERSION}/helmfile_${HELMFILE_VERSION}_linux_amd64.tar.gz"

# Extraer
tar -xzf helmfile_${HELMFILE_VERSION}_linux_amd64.tar.gz

# Mover a PATH
sudo mv helmfile /usr/local/bin/helmfile
sudo chmod +x /usr/local/bin/helmfile

# Limpiar
rm helmfile_${HELMFILE_VERSION}_linux_amd64.tar.gz

# Verificar
helmfile --version
```

**Salida esperada:**
```
helmfile version v0.159.0
```

#### Arch/Manjaro (alternativa con AUR)
```bash
# Si usas un AUR helper como yay
yay -S helmfile
```

### 5. Kind (Kubernetes in Docker)

Kind crea clusters de Kubernetes usando contenedores Docker.

#### Todos los sistemas
```bash
# Descargar binario
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

# Hacer ejecutable
chmod +x ./kind

# Mover a PATH
sudo mv ./kind /usr/local/bin/kind

# Verificar
kind --version
```

**Salida esperada:**
```
kind version 0.20.0
```

### 6. Direnv

Direnv carga automáticamente variables de entorno al entrar en un directorio.

#### Arch/Manjaro
```bash
sudo pacman -S direnv

# Agregar hook a tu shell
# Para bash:
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc

# Para zsh:
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
source ~/.zshrc

# Verificar
direnv --version
```

#### Debian/Ubuntu
```bash
sudo apt install -y direnv

# Agregar hook
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

**Salida esperada:**
```
2.32.3
```

### 7. Git

```bash
# Arch/Manjaro
sudo pacman -S git

# Debian/Ubuntu
sudo apt install -y git

# Configurar (si no lo has hecho)
git config --global user.name "Tu Nombre"
git config --global user.email "tu@email.com"

# Verificar
git --version
```

### 8. Herramientas Opcionales (pero útiles)

```bash
# jq - Para parsear JSON
sudo pacman -S jq

# yq - Para parsear YAML
sudo pacman -S yq

# tree - Para visualizar estructura de directorios
sudo pacman -S tree

# curl y wget (probablemente ya instalados)
sudo pacman -S curl wget

# bat - cat con syntax highlighting (opcional)
sudo pacman -S bat

# htop - Monitor de recursos
sudo pacman -S htop
```

## 🚀 Crear Cluster de Kubernetes

### Configuración de Kind

Vamos a crear un cluster Kind optimizado para desarrollo local:

```bash
# Crear archivo de configuración
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: helmfile-tutorial
nodes:
  - role: control-plane
    extraPortMappings:
      # Puerto para Nginx Ingress
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
EOF
```

**Explicación de la configuración:**
- `extraPortMappings`: Mapea puertos del contenedor al host (necesario para Ingress)
- `node-labels`: Label para que Nginx Ingress use este nodo

### Crear el cluster

```bash
# Crear cluster con la configuración
kind create cluster --config kind-config.yaml

# Verificar que el cluster está corriendo
kubectl cluster-info --context kind-helmfile-tutorial
```

**Salida esperada:**
```
Kubernetes control plane is running at https://127.0.0.1:xxxxx
CoreDNS is running at https://127.0.0.1:xxxxx/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### Verificar nodos

```bash
kubectl get nodes
```

**Salida esperada:**
```
NAME                              STATUS   ROLES           AGE   VERSION
helmfile-tutorial-control-plane   Ready    control-plane   1m    v1.27.3
```

### Crear namespaces para los ambientes

```bash
# Crear namespaces
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Verificar
kubectl get namespaces
```

**Salida esperada:**
```
NAME              STATUS   AGE
default           Active   2m
dev               Active   10s
staging           Active   10s
production        Active   10s
kube-system       Active   2m
kube-public       Active   2m
kube-node-lease   Active   2m
```

## 📁 Clonar el Repositorio

```bash
# Clonar el repositorio del tutorial
git clone https://gitlab.com/matias-tecnosoul/helmfile-microservices.git
cd helmfile-microservices

# Ver estructura
tree -L 2 -I 'node_modules'
```

**Estructura esperada:**
```
helmfile-microservices/
├── helmfile.yaml
├── helmfile.d/
│   ├── 01-infrastructure.yaml
│   ├── 02-services.yaml
│   ├── 03-ingress.yaml
│   ├── environments/
│   └── values/
├── charts/
│   ├── api-gateway/
│   ├── auth-service/
│   └── user-service/
├── scripts/
│   ├── setup-cluster.sh
│   ├── verify.sh
│   └── cleanup.sh
├── .envrc-sample
├── README.md
└── docs/
```

## 🔧 Configurar Direnv

### Crear archivo .envrc

```bash
# Copiar template
cp .envrc-sample .envrc

# Editar valores
nano .envrc
```

**Contenido de .envrc:**
```bash
# Configuración de Kubernetes
export KUBECONFIG=$PWD/.kube/config
export KUBE_CONTEXT=kind-helmfile-tutorial

# Ambiente por defecto
export HELMFILE_ENVIRONMENT=dev

# Opcional: Habilitar debug de helmfile
# export HELMFILE_DEBUG=1

# Opcional: Cache de Helm
export HELM_CACHE_HOME=$PWD/.helm/cache
export HELM_CONFIG_HOME=$PWD/.helm/config
export HELM_DATA_HOME=$PWD/.helm/data
```

### Permitir direnv

```bash
# Direnv pedirá permiso explícito
direnv allow

# Deberías ver un mensaje como:
# direnv: loading ~/helmfile-microservices/.envrc
# direnv: export +HELM_CACHE_HOME +HELM_CONFIG_HOME +HELM_DATA_HOME +HELMFILE_ENVIRONMENT +KUBE_CONTEXT +KUBECONFIG
```

### Verificar que las variables se cargaron

```bash
# Salir y entrar al directorio
cd ..
cd helmfile-microservices

# Verificar variables
echo $KUBECONFIG
echo $KUBE_CONTEXT
echo $HELMFILE_ENVIRONMENT
```

**Salida esperada:**
```
/home/tu-usuario/helmfile-microservices/.kube/config
kind-helmfile-tutorial
dev
```

## ✅ Verificación Final

Ejecuta este script para verificar que todo está instalado correctamente:

```bash
cat > verify-setup.sh << 'EOF'
#!/bin/bash

echo "=== Verificando instalaciones ==="
echo ""

check_command() {
    if command -v $1 &> /dev/null; then
        version=$($1 --version 2>&1 | head -n 1)
        echo "✅ $1: $version"
        return 0
    else
        echo "❌ $1 NO instalado"
        return 1
    fi
}

check_command docker
check_command kubectl
check_command helm
check_command helmfile
check_command kind
check_command git
check_command direnv
check_command jq
check_command yq

echo ""
echo "=== Verificando plugins de Helm ==="
if helm plugin list | grep -q "diff"; then
    echo "✅ helm-diff instalado"
else
    echo "❌ helm-diff NO instalado"
fi

if helm plugin list | grep -q "secrets"; then
    echo "✅ helm-secrets instalado"
else
    echo "❌ helm-secrets NO instalado"
fi

echo ""
echo "=== Verificando cluster de Kubernetes ==="
if kubectl cluster-info &> /dev/null; then
    echo "✅ Cluster de Kubernetes accesible"
    kubectl get nodes
else
    echo "❌ No hay cluster de Kubernetes corriendo"
    echo "   Ejecuta: kind create cluster --config kind-config.yaml"
fi

echo ""
echo "=== Verificando namespaces ==="
for ns in dev staging production; do
    if kubectl get namespace $ns &> /dev/null; then
        echo "✅ Namespace $ns existe"
    else
        echo "❌ Namespace $ns NO existe"
        echo "   Ejecuta: kubectl create namespace $ns"
    fi
done

echo ""
echo "=== Verificando Docker ==="
if docker ps &> /dev/null; then
    echo "✅ Docker corriendo correctamente"
else
    echo "❌ Docker no está corriendo o faltan permisos"
    echo "   Ejecuta: sudo systemctl start docker"
    echo "   Y agrega tu usuario al grupo: sudo usermod -aG docker $USER"
fi

echo ""
echo "=== Verificando Direnv ==="
if [ -n "$KUBECONFIG" ]; then
    echo "✅ Direnv cargó variables de entorno"
    echo "   KUBECONFIG: $KUBECONFIG"
    echo "   KUBE_CONTEXT: $KUBE_CONTEXT"
    echo "   HELMFILE_ENVIRONMENT: $HELMFILE_ENVIRONMENT"
else
    echo "⚠️  Direnv no cargó variables"
    echo "   Ejecuta: direnv allow"
fi

echo ""
echo "=== Resumen ==="
echo "Si todos los checks están en ✅, estás listo para continuar."
echo "Caso contrario, revisa los pasos anteriores."
EOF

chmod +x verify-setup.sh
./verify-setup.sh
```

**Salida esperada (todo OK):**
```
=== Verificando instalaciones ===

✅ docker: Docker version 24.0.7, build afdd53b
✅ kubectl: Client Version: v1.28.0
✅ helm: version.BuildInfo{Version:"v3.13.0"}
✅ helmfile: helmfile version v0.159.0
✅ kind: kind version 0.20.0
✅ git: git version 2.42.0
✅ direnv: 2.32.3
✅ jq: jq-1.7
✅ yq: yq (https://github.com/mikefarah/yq/) version v4.35.2

=== Verificando plugins de Helm ===
✅ helm-diff instalado
✅ helm-secrets instalado

=== Verificando cluster de Kubernetes ===
✅ Cluster de Kubernetes accesible
NAME                              STATUS   ROLES           AGE   VERSION
helmfile-tutorial-control-plane   Ready    control-plane   5m    v1.27.3

=== Verificando namespaces ===
✅ Namespace dev existe
✅ Namespace staging existe
✅ Namespace production existe

=== Verificando Docker ===
✅ Docker corriendo correctamente

=== Verificando Direnv ===
✅ Direnv cargó variables de entorno
   KUBECONFIG: /home/usuario/helmfile-microservices/.kube/config
   KUBE_CONTEXT: kind-helmfile-tutorial
   HELMFILE_ENVIRONMENT: dev

=== Resumen ===
Si todos los checks están en ✅, estás listo para continuar.
Caso contrario, revisa los pasos anteriores.
```

## 🔧 Configuración Adicional

### Aliases útiles

Agrega estos aliases a tu `~/.bashrc` o `~/.zshrc`:

```bash
# Kubernetes
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgi='kubectl get ingress'
alias kl='kubectl logs'
alias kd='kubectl describe'
alias kx='kubectl exec -it'

# Helm
alias h='helm'
alias hl='helm list'
alias hi='helm install'
alias hu='helm upgrade'

# Helmfile
alias hf='helmfile'
alias hfa='helmfile apply'
alias hfs='helmfile sync'
alias hfd='helmfile diff'
alias hfl='helmfile list'
alias hft='helmfile template'

# Kind
alias kc='kind create cluster --config'
alias kdel='kind delete cluster --name'
```

Luego recarga:
```bash
source ~/.bashrc  # o ~/.zshrc
```

### Configurar autocompletado

```bash
# kubectl
kubectl completion bash > ~/.kubectl-completion
echo 'source ~/.kubectl-completion' >> ~/.bashrc

# helm
helm completion bash > ~/.helm-completion
echo 'source ~/.helm-completion' >> ~/.bashrc

# Recargar
source ~/.bashrc
```

## 🛠️ Scripts Útiles

### Script de setup completo

Ya incluido en el repo como `scripts/setup-cluster.sh`:

```bash
#!/bin/bash
set -e

echo "🚀 Setup de cluster Kind para tutorial Helmfile"
echo ""

# Verificar que Kind está instalado
if ! command -v kind &> /dev/null; then
    echo "❌ Kind no está instalado. Por favor instala Kind primero."
    exit 1
fi

# Crear cluster si no existe
if ! kind get clusters | grep -q "helmfile-tutorial"; then
    echo "📦 Creando cluster Kind..."
    kind create cluster --config kind-config.yaml
else
    echo "✅ Cluster Kind ya existe"
fi

# Crear namespaces
echo "📁 Creando namespaces..."
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -

# Verificar
echo ""
echo "✅ Setup completo"
echo ""
kubectl get nodes
echo ""
kubectl get namespaces
```

### Script de cleanup

Incluido como `scripts/cleanup.sh`:

```bash
#!/bin/bash
set -e

echo "🧹 Limpiando ambiente..."

# Eliminar cluster
if kind get clusters | grep -q "helmfile-tutorial"; then
    echo "🗑️  Eliminando cluster Kind..."
    kind delete cluster --name helmfile-tutorial
else
    echo "ℹ️  Cluster no existe"
fi

# Limpiar cache de Helm
if [ -d ".helm" ]; then
    echo "🗑️  Limpiando cache de Helm..."
    rm -rf .helm
fi

echo "✅ Cleanup completo"
```

## 🐛 Troubleshooting

### Docker: permission denied

```bash
# Si ves este error al ejecutar docker:
# Got permission denied while trying to connect to the Docker daemon socket

# Solución:
sudo usermod -aG docker $USER
newgrp docker

# O cierra sesión y vuelve a entrar
```

### Kind: failed to create cluster

```bash
# Si Kind falla al crear el cluster:

# 1. Verificar que Docker está corriendo
docker ps

# 2. Limpiar clusters viejos
kind delete cluster --name helmfile-tutorial

# 3. Verificar espacio en disco
df -h

# 4. Ver logs detallados
kind create cluster --config kind-config.yaml --verbosity=3
```

### kubectl: connection refused

```bash
# Si kubectl no puede conectar al cluster:

# 1. Verificar que el cluster existe
kind get clusters

# 2. Verificar contexto actual
kubectl config current-context

# 3. Cambiar a contexto correcto
kubectl config use-context kind-helmfile-tutorial

# 4. Verificar que KUBECONFIG apunta al archivo correcto
echo $KUBECONFIG
```

### Direnv: variables no se cargan

```bash
# Si las variables de entorno no se cargan automáticamente:

# 1. Verificar que direnv está en el hook del shell
# Para bash:
grep direnv ~/.bashrc

# Debería mostrar:
# eval "$(direnv hook bash)"

# 2. Si no está, agregarlo:
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc

# 3. Permitir direnv en el directorio
cd ~/helmfile-microservices
direnv allow

# 4. Salir y entrar al directorio para forzar recarga
cd ..
cd helmfile-microservices
```

### Helmfile: command not found

```bash
# Si helmfile no se encuentra en el PATH:

# 1. Verificar instalación
which helmfile

# 2. Si no está, verificar que /usr/local/bin está en PATH
echo $PATH | grep /usr/local/bin

# 3. Si no está, agregarlo:
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc

# 4. Reinstalar helmfile si es necesario
# (ver sección de instalación de Helmfile arriba)
```

### Puerto 80/443 ya en uso

```bash
# Si al crear el cluster Kind dice que los puertos están ocupados:

# 1. Ver qué proceso usa el puerto
sudo lsof -i :80
sudo lsof -i :443

# 2. Si es otro servicio (nginx, apache), detenerlo temporalmente:
sudo systemctl stop nginx
sudo systemctl stop apache2

# 3. O modificar kind-config.yaml para usar otros puertos:
# Cambiar hostPort: 80 por hostPort: 8080
# Cambiar hostPort: 443 por hostPort: 8443
```

## ✅ Checklist Final

Antes de continuar al siguiente capítulo, verifica:

- [ ] Docker instalado y corriendo
- [ ] kubectl instalado
- [ ] Helm instalado con plugins (diff, secrets)
- [ ] Helmfile instalado
- [ ] Kind instalado
- [ ] Direnv instalado y configurado
- [ ] Cluster Kind creado y funcionando
- [ ] Namespaces dev/staging/production creados
- [ ] Variables de entorno cargadas con direnv
- [ ] Script verify-setup.sh muestra todo ✅
- [ ] Puedes ejecutar: `helmfile --version` sin errores

## 📚 Comandos de Referencia Rápida

```bash
# Ver estado del cluster
kubectl cluster-info
kubectl get nodes

# Ver namespaces
kubectl get namespaces

# Ver contextos de kubectl
kubectl config get-contexts

# Cambiar contexto
kubectl config use-context kind-helmfile-tutorial

# Ver releases de Helm en un namespace
helm list -n dev

# Ver versiones de herramientas
docker --version
kubectl version --client
helm version
helmfile --version
kind --version
```

## ➡️ Siguiente Paso

Una vez que tengas todo instalado y verificado, continúa con:

👉 **[02 - Introducción a Helmfile](02-intro-helmfile.md)**

En el próximo capítulo:
- Entenderás qué es Helmfile y cuándo usarlo
- Crearás tu primer helmfile.yaml
- Desplegarás PostgreSQL con Helmfile
- Compararás Helmfile vs Helm manual

## 📚 Referencias

- [Docker Documentation](https://docs.docker.com/)
- [kubectl Documentation](https://kubernetes.io/docs/reference/kubectl/)
- [Helm Documentation](https://helm.sh/docs/)
- [Helmfile Documentation](https://helmfile.readthedocs.io/)
- [Kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Direnv Documentation](https://direnv.net/)

---

**💡 Tip**: Guarda este documento como referencia. Los comandos de troubleshooting son útiles durante todo el tutorial.