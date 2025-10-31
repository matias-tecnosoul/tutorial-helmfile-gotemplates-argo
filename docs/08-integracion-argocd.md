# 08 - Integración ArgoCD (45 min)

## 🎯 Objetivo

Implementar GitOps con ArgoCD, usando Helmfile como generador de manifiestos y ArgoCD como motor de sincronización automática.

## 📝 GitOps con Helmfile + ArgoCD

**Flujo tradicional:**
```bash
Developer → helmfile apply → Kubernetes
# Requiere acceso directo al cluster
```

**Flujo GitOps:**
```bash
Developer → git push → ArgoCD detecta cambio → Kubernetes
# ArgoCD sincroniza automáticamente
```

## 🎨 Dos Approaches

### Approach 1: Manifiestos Pre-generados (Recomendado)

```
helmfile template → manifests/ → git push → ArgoCD sync
```

**Ventajas:**
- Simple de entender
- ArgoCD solo aplica YAML
- Fácil de debuggear
- No requiere plugin de Helmfile

### Approach 2: ArgoCD ejecuta Helmfile (Avanzado)

```
git push → ArgoCD ejecuta helmfile → Kubernetes
```

**Ventajas:**
- Menos archivos en Git
- Templates dinámicos en runtime

**Desventajas:**
- Requiere plugin custom
- Más complejo de debuggear

**Usaremos Approach 1** para este tutorial.

## 🚀 Instalar ArgoCD

### Deploy de ArgoCD

```bash
# Crear namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que todos los pods estén ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

# Verificar
kubectl get pods -n argocd
```

**Salida esperada:**
```
NAME                                  READY   STATUS    AGE
argocd-server-xxx                     1/1     Running   2m
argocd-repo-server-xxx                1/1     Running   2m
argocd-application-controller-xxx     1/1     Running   2m
argocd-dex-server-xxx                 1/1     Running   2m
argocd-redis-xxx                      1/1     Running   2m
```

### Acceder a ArgoCD UI

```bash
# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Obtener password inicial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Abrir navegador
firefox https://localhost:8080
# User: admin
# Pass: (del comando anterior)
```

### Instalar ArgoCD CLI (Opcional)

```bash
# Arch/Manjaro
sudo pacman -S argocd

# Debian/Ubuntu
curl -sSL -o /tmp/argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 /tmp/argocd-linux-amd64 /usr/local/bin/argocd

# Login via CLI
argocd login localhost:8080 --insecure
# User: admin
# Pass: (del secret)

# Verificar
argocd version
```

## 📁 Estructura GitOps

```bash
# Crear directorio para manifiestos generados
mkdir -p manifests/{dev,staging,production}
```

**Estructura actualizada:**
```
helmfile-microservices/
├── helmfile.yaml
├── helmfile.d/
├── charts/
├── manifests/              # NUEVO: Manifiestos generados
│   ├── dev/
│   ├── staging/
│   └── production/
└── argocd/                 # NUEVO: Configuración de ArgoCD
    ├── application-dev.yaml
    ├── application-staging.yaml
    └── application-production.yaml
```

## 🔄 Generar Manifiestos

### Script de generación

```bash
# scripts/generate-manifests.sh
#!/bin/bash

set -e

ENVIRONMENTS=("dev" "staging" "production")

echo "🔨 Generando manifiestos con Helmfile..."
echo ""

for env in "${ENVIRONMENTS[@]}"; do
  echo "📦 Ambiente: $env"
  
  # Limpiar directorio
  rm -rf manifests/$env/*
  mkdir -p manifests/$env
  
  # Generar manifiestos
  helmfile -e $env template --output-dir manifests/$env
  
  echo "✅ $env completado"
  echo ""
done

echo "✅ Todos los manifiestos generados"
```

```bash
chmod +x scripts/generate-manifests.sh
./scripts/generate-manifests.sh
```

**Salida esperada:**
```
manifests/
├── dev/
│   ├── ingress-nginx/
│   │   └── ingress-nginx/
│   │       └── templates/
│   └── dev/
│       ├── auth-service/
│       ├── user-service/
│       ├── api-gateway/
│       ├── postgres/
│       └── redis/
├── staging/
└── production/
```

### .gitignore (actualizar)

```bash
# .gitignore

# Secrets
helmfile.d/environments/*/secrets.yaml
!helmfile.d/environments/*/secrets.yaml.example

# Local
.envrc
.kube/
.helm/

# Manifiestos generados (COMENTAR para commitear)
# manifests/
```

**Nota:** Para GitOps, **SÍ committear** manifests/.

## 🎯 ArgoCD Applications

### argocd/application-dev.yaml

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helmfile-microservices-dev
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo
    targetRevision: main
    path: manifests/dev
  
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  
  syncPolicy:
    automated:
      prune: true      # Eliminar recursos huérfanos
      selfHeal: true   # Auto-corregir drift
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### argocd/application-staging.yaml

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helmfile-microservices-staging
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo
    targetRevision: main
    path: manifests/staging
  
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### argocd/application-production.yaml

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helmfile-microservices-production
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo
    targetRevision: main
    path: manifests/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    # Producción: sync MANUAL (más control)
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## 🔗 Conectar ArgoCD con Git

### Opción 1: Repositorio público

```bash
# ArgoCD puede acceder sin credenciales
# Solo crear la Application
kubectl apply -f argocd/application-dev.yaml
```

### Opción 2: Repositorio privado

```bash
# Agregar repositorio con credenciales
argocd repo add https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo \
  --username YOUR_USERNAME \
  --password YOUR_TOKEN

# O via UI: Settings → Repositories → Connect Repo
```

### Crear Applications

```bash
# Dev (sync automático)
kubectl apply -f argocd/application-dev.yaml

# Staging (sync automático)
kubectl apply -f argocd/application-staging.yaml

# Production (sync manual)
kubectl apply -f argocd/application-production.yaml

# Verificar
kubectl get applications -n argocd
```

## 🎮 Workflow GitOps Completo

### 1. Desarrollo local

```bash
# Hacer cambios
nano helmfile.d/values/common.yaml

# Probar localmente
helmfile -e dev diff
helmfile -e dev apply
```

### 2. Generar manifiestos

```bash
# Generar para todos los ambientes
./scripts/generate-manifests.sh

# Verificar cambios
git status
git diff manifests/
```

### 3. Commit y push

```bash
# Agregar cambios
git add helmfile.d/ manifests/
git commit -m "feat: aumentar recursos de postgres en production"
git push origin main
```

### 4. ArgoCD sincroniza automáticamente

```bash
# Dev y staging se sincronizan automáticamente
# Production requiere sync manual

# Ver status via CLI
argocd app get helmfile-microservices-dev

# O en UI: https://localhost:8080
```

### 5. Sync manual en production

```bash
# Via CLI
argocd app sync helmfile-microservices-production

# O via UI: Click en "Sync" button
```

## 📊 Monitoring en ArgoCD UI

### Ver estado de aplicaciones

**UI muestra:**
- 🟢 **Synced**: Git == Cluster
- 🟡 **OutOfSync**: Git ≠ Cluster
- 🔴 **Failed**: Error al sincronizar
- ⏳ **Progressing**: Sincronizando...

### Ver diferencias (Diff)

```bash
# Via CLI
argocd app diff helmfile-microservices-dev

# Via UI: App → Diff tab
```

### Ver recursos desplegados

**UI muestra árbol de recursos:**
```
Application
├── Deployment: auth-service
├── Deployment: user-service
├── Service: auth-service
├── Service: user-service
├── Ingress: auth-service
└── StatefulSet: postgres
```

## 🔄 Casos de Uso

### Rollback

```bash
# Ver historial
argocd app history helmfile-microservices-dev

# Rollback a revisión anterior
argocd app rollback helmfile-microservices-dev 2

# O via UI: History → Rollback
```

### Manual Sync con opciones

```bash
# Sync forzado (reemplaza todo)
argocd app sync helmfile-microservices-dev --force

# Sync solo recursos específicos
argocd app sync helmfile-microservices-dev \
  --resource apps:Deployment:auth-service

# Dry-run (preview)
argocd app sync helmfile-microservices-dev --dry-run
```

### Deshabilitar auto-sync temporalmente

```bash
# Deshabilitar
argocd app set helmfile-microservices-dev --sync-policy none

# Re-habilitar
argocd app set helmfile-microservices-dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## 🛡️ Best Practices GitOps

### 1. Branches por ambiente

```bash
# Estrategia avanzada
main → production (sync manual)
staging → staging (sync auto)
develop → dev (sync auto)
```

### 2. PR reviews obligatorios

```yaml
# .github/workflows/validate.yml
name: Validate Manifests

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Validate YAML
        run: |
          find manifests/ -name "*.yaml" -exec yamllint {} \;
```

### 3. Generar manifiestos en CI

```yaml
# .gitlab-ci.yml
generate-manifests:
  stage: build
  script:
    - ./scripts/generate-manifests.sh
  artifacts:
    paths:
      - manifests/
```

### 4. Secrets management

```bash
# NO committear secrets en manifests/
# Usar sealed-secrets o external-secrets

# Ejemplo con sealed-secrets
kubeseal < secret.yaml > sealed-secret.yaml
git add sealed-secret.yaml  # Safe to commit
```

## 🐛 Troubleshooting

### Application OutOfSync constante

```bash
# Verificar drift
argocd app diff helmfile-microservices-dev

# Causa común: changes manuales con kubectl
# Solución: selfHeal: true en syncPolicy
```

### Sync falla con error

```bash
# Ver logs detallados
argocd app logs helmfile-microservices-dev

# Ver eventos
kubectl get events -n dev --sort-by='.lastTimestamp'

# Describir aplicación
argocd app get helmfile-microservices-dev
```

### Manifiestos no se actualizan

```bash
# Forzar refresh de repo
argocd app get helmfile-microservices-dev --refresh

# Verificar que los manifiestos están en Git
git log --oneline manifests/dev/
```

### ArgoCD no puede acceder al repo

```bash
# Verificar conexión
argocd repo list

# Re-agregar repo
argocd repo add https://github.com/USER/REPO \
  --username USER \
  --password TOKEN
```

## 📝 Helmfile vs ArgoCD: Responsabilidades

| Responsabilidad | Helmfile | ArgoCD |
|----------------|----------|--------|
| **Templating** | ✅ Go templates | ❌ |
| **Multi-ambiente** | ✅ environments | ❌ |
| **Generar manifiestos** | ✅ helmfile template | ❌ |
| **Aplicar al cluster** | ⚠️ Opcional | ✅ |
| **Detectar drift** | ❌ | ✅ |
| **Auto-sync** | ❌ | ✅ |
| **Rollback** | ⚠️ helm rollback | ✅ |
| **Multi-cluster** | ⚠️ kubeContext | ✅ |
| **GitOps** | ❌ | ✅ |

## ✅ Checklist

- [ ] ArgoCD instalado y accesible via UI
- [ ] Script generate-manifests.sh funciona
- [ ] Manifiestos commiteados en Git
- [ ] Applications de ArgoCD creadas
- [ ] Dev/Staging con sync automático
- [ ] Production con sync manual
- [ ] Probaste cambio → commit → sync
- [ ] Entiendes flujo GitOps completo

## 🎓 Recursos Adicionales

### ArgoCD

- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

### GitOps

- [GitOps Principles](https://opengitops.dev/)
- [Weaveworks GitOps Guide](https://www.weave.works/technologies/gitops/)

## 🎉 Tutorial Completado

¡Felicitaciones! Has completado el tutorial de Helmfile + Go Templates + ArgoCD.

**Lo que aprendiste:**
1. ✅ Setup de herramientas (Helmfile, Kind, Direnv)
2. ✅ Helmfile básico y avanzado
3. ✅ Go Templates (variables, condicionales, loops, pipelines)
4. ✅ Multi-ambiente (dev/staging/production)
5. ✅ Helmfile modular (helmfile.d/)
6. ✅ Dependencias (needs, wait, timeout)
7. ✅ Ingress (Nginx, hosts dinámicos)
8. ✅ GitOps con ArgoCD

**Stack desplegado:**
- 📦 PostgreSQL
- 📦 Redis
- 🚀 Auth Service
- 🚀 User Service
- 🚀 API Gateway
- 🌐 Nginx Ingress Controller
- 🔄 ArgoCD

**Próximos pasos:**
- 🔐 Tutorial de Secrets (SOPS/Sealed Secrets)
- 📊 Monitoring con Prometheus/Grafana
- 🔒 Service Mesh (Istio/Linkerd)
- 🌍 Multi-cluster con ArgoCD

---

**💡 Tip final**: GitOps es poderoso pero requiere disciplina. Nunca hagas `kubectl apply` manual en ambientes gestionados por ArgoCD.

**📧 Feedback**: Si encuentras errores o mejoras, abre un issue en: https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo/issues