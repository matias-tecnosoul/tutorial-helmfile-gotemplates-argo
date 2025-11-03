# Tutorial Helmfile + Go Templates + ArgoCD

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Tutorial completo y práctico para gestionar múltiples microservices en Kubernetes usando Helmfile, Go Templates y GitOps con ArgoCD.

## 🎯 Objetivos de Aprendizaje

Al completar este tutorial, aprenderás a:

- ✅ Gestionar múltiples aplicaciones con **Helmfile**
- ✅ Crear configuraciones dinámicas con **Go Templates**
- ✅ Organizar proyectos con **helmfile.d/** (patrón modular)
- ✅ Gestionar múltiples ambientes (dev/staging/production)
- ✅ Manejar dependencias entre servicios
- ✅ Exponer aplicaciones con **Nginx Ingress**
- ✅ Implementar **GitOps** con ArgoCD

## 📋 Requisitos Previos

### Conocimientos

- Bash/Terminal básico
- Git básico
- Conceptos de Docker y Kubernetes
- Helm básico (recomendado)

### Software Necesario

| Herramienta | Versión | Instalación |
|-------------|---------|-------------|
| Docker      | 20+     | [docs.docker.com](https://docs.docker.com/get-docker/) |
| kubectl     | 1.28+   | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) |
| Helm        | 3.12+   | [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) |
| Helmfile    | 1.0+    | [helmfile.readthedocs.io](https://helmfile.readthedocs.io/en/latest/#installation) |
| Kind        | 0.20+   | [kind.sigs.k8s.io/docs/user/quick-start](https://kind.sigs.k8s.io/docs/user/quick-start/) |
| Asdf        | 0.13+   | [asdf-vm.com](https://asdf-vm.com/guide/getting-started.html) (opcional) |
| Direnv      | 2.32+   | [direnv.net](https://direnv.net/docs/installation.html) (opcional) |

## 🚀 Quick Start

### Opción 1: Deploy local en 10 minutos

```bash
# 1. Clonar el repositorio
git clone https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo.git
cd tutorial-helmfile-gotemplates-argo

# 2. Configurar direnv (opcional)
cp .envrc-sample .envrc
direnv allow

# 3. Crear cluster Kind
kind create cluster --config kind-config.yaml

# 4. Crear namespace
kubectl create namespace dev

# 5. Desplegar infraestructura en dev
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# 6. Verificar
kubectl get all -n dev
```

## 🗂️ Estructura del Proyecto

```
tutorial-helmfile-gotemplates-argo/
├── helmfile.d/                      # Helmfiles modulares (patrón Mikroways)
│   ├── 01-infrastructure.yaml       # PostgreSQL, Redis
│   ├── 02-services.yaml             # Microservices
│   ├── 03-ingress.yaml              # Nginx Ingress
│   ├── environments/                # Configuración por ambiente
│   │   ├── dev/
│   │   │   ├── values.yaml
│   │   │   ├── secrets.yaml
│   │   │   └── secrets.yaml.example
│   │   ├── staging/
│   │   └── production/
│   └── values/                      # Values por componente
│       ├── common.yaml
│       ├── postgres/
│       │   └── values.yaml.gotmpl   # ← Con Go Templates
│       ├── redis/
│       ├── auth-service/
│       ├── user-service/
│       ├── api-gateway/
│       └── nginx-ingress/
├── charts/                          # Charts custom
│   ├── auth-service/
│   ├── user-service/
│   └── api-gateway/
├── manifests/                       # Manifiestos generados (GitOps)
│   ├── dev/
│   ├── staging/
│   └── production/
├── argocd/                          # Configuración de ArgoCD
│   ├── application-dev.yaml
│   ├── application-staging.yaml
│   └── application-production.yaml
├── scripts/
│   ├── setup-cluster.sh
│   ├── generate-manifests.sh
│   ├── test-endpoints.sh
│   └── cleanup.sh
└── docs/                            # Documentación paso a paso
    ├── 01-setup.md
    ├── 02-intro-helmfile.md
    ├── 03-go-templates.md
    ├── 04-multi-env.md
    ├── 05-helmfile-modular.md
    ├── 06-dependencies.md
    ├── 07-ingress.md
    └── 08-integracion-argocd.md
```

> 💡 **Patrón Mikroways**: Este tutorial NO usa `helmfile.yaml` en la raíz. 
> Cada módulo en `helmfile.d/` se ejecuta independientemente, permitiendo deploy selectivo.

## 📚 Documentación

### Guías Paso a Paso

1. [**Setup del Entorno**](docs/01-setup.md) (30 min)
   * Instalación de herramientas
   * Configuración de Kind cluster
   * Verificación del ambiente

2. [**Introducción a Helmfile**](docs/02-intro-helmfile.md) (45 min)
   * Qué es Helmfile y por qué usarlo
   * Tu primer helmfile modular
   * Deploy de PostgreSQL

3. [**Go Templates**](docs/03-go-templates.md) (1h)
   * Variables y acceso a valores
   * Flujo de carga de valores (common → env → secrets)
   * Condicionales (if/else)
   * Loops (range)
   * Pipelines y funciones
   * With para reducir repetición

4. [**Multi-Ambiente**](docs/04-multi-env.md) (45 min)
   * Estructura de environments/
   * Herencia de valores (common → dev → secrets)
   * Gestión de secrets (sin SOPS)
   * Deploy por ambiente

5. [**Helmfile Modular**](docs/05-helmfile-modular.md) (45 min)
   * Patrón helmfile.d/
   * Organización por categoría
   * Deploy selectivo
   * Best practices de Mikroways

6. [**Dependencias**](docs/06-dependencies.md) (30 min)
   * needs: entre releases
   * wait y timeout
   * Orden de ejecución
   * Dependencias condicionales

7. [**Ingress**](docs/07-ingress.md) (45-60 min)
   * Nginx Ingress Controller
   * Ingress resources templating
   * Hosts dinámicos por ambiente
   * Testing de endpoints

8. [**Integración ArgoCD**](docs/08-integracion-argocd.md) (45 min)
   * GitOps workflow
   * Instalación de ArgoCD
   * Generación de manifiestos
   * Sync automático y manual

**Tiempo total estimado: ~6 horas**

## 🏗️ Arquitectura

### Stack Desplegado

```
                  ┌─────────────┐
                  │ API Gateway │
                  └──────┬──────┘
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
    ┌──────────┐  ┌─────────────┐  ┌──────────┐
    │   Auth   │  │    User     │  │  Redis   │
    │ Service  │  │   Service   │  │          │
    └────┬─────┘  └──────┬──────┘  └──────────┘
         │               │
         └───────┬───────┘
                 ↓
         ┌─────────────┐
         │ PostgreSQL  │
         └─────────────┘
                 ↑
         ┌───────┴───────┐
         │ Nginx Ingress │
         └───────────────┘
```

### Flujo GitOps

```
Developer
    ↓
git push
    ↓
helmfile template → manifests/
    ↓
ArgoCD detecta cambio
    ↓
Sync a Kubernetes
```

## 🎮 Comandos Útiles

```bash
# Alias útil (opcional)
alias hf-infra='helmfile -f helmfile.d/01-infrastructure.yaml'
alias hf-services='helmfile -f helmfile.d/02-services.yaml'
alias hf-ingress='helmfile -f helmfile.d/03-ingress.yaml'

# Helmfile - Infraestructura
hf-infra -e dev list              # Listar releases
hf-infra -e dev diff              # Ver diferencias
hf-infra -e dev apply             # Aplicar cambios
hf-infra -e dev destroy           # Eliminar todo
hf-infra -e dev template          # Ver manifiestos generados
hf-infra -e dev write-values      # Ver valores mergeados

# Deploy selectivo por labels
hf-infra -e dev -l component=database apply
hf-infra -e dev -l component=cache apply
hf-infra -e dev -l tier=infrastructure apply

# Kubernetes
kubectl get all -n dev
kubectl logs -n dev -l app.kubernetes.io/name=postgres
kubectl describe pod -n dev <pod-name>

# Ingress testing
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
curl -H "Host: auth.dev.example.local" http://localhost:8080/

# ArgoCD
argocd app list
argocd app get helmfile-microservices-dev
argocd app sync helmfile-microservices-dev
```

## 🔐 Secrets Management

⚠️ **IMPORTANTE**: Este tutorial usa secrets en plain text para simplificar el aprendizaje.

**En producción REAL:**

1. **NUNCA** committees `secrets.yaml` sin cifrar
2. Usa SOPS, Sealed Secrets, o External Secrets
3. Ver tutorial futuro sobre secrets management

```bash
# .gitignore
helmfile.d/environments/*/secrets.yaml
!helmfile.d/environments/*/secrets.yaml.example
```

## 🧪 Testing

```bash
# Probar endpoints localmente
./scripts/test-endpoints.sh

# Verificar deploy
kubectl get all -n dev

# Ver logs de un servicio
kubectl logs -n dev -l app.kubernetes.io/name=postgres -f

# Port-forward a un servicio
kubectl port-forward -n dev svc/postgres 5432:5432
```

## 🔄 Workflow de Desarrollo

### 1. Desarrollo local

```bash
# Hacer cambios
nano helmfile.d/values/common.yaml

# Probar localmente
helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
```

### 2. Commit y push

```bash
# Generar manifiestos
./scripts/generate-manifests.sh

# Commit
git add helmfile.d/ manifests/
git commit -m "feat: aumentar recursos de postgres"
git push origin main
```

### 3. Deploy automático (GitOps)

```bash
# ArgoCD sincroniza automáticamente dev/staging
# Production requiere sync manual

# Verificar en ArgoCD UI
firefox https://localhost:8080
```

## 📊 Comparación: Helm vs Helmfile

| Aspecto | Helm Manual | Helmfile |
|---------|-------------|----------|
| **Múltiples apps** | Scripts bash | Declarativo YAML |
| **Ambientes** | Múltiples commands | `-e prod` |
| **Dependencias** | Manual | `needs:` built-in |
| **Diff** | Plugin separado | `helmfile diff` |
| **Templating** | Helm templates | Go templates + Helm |
| **State** | Solo en cluster | Versionable en Git |
| **Deploy selectivo** | Uno por uno | Labels y filtros |

## 🛠️ Scripts Incluidos

| Script | Descripción |
|--------|-------------|
| `setup-cluster.sh` | Crea cluster Kind y namespaces |
| `generate-manifests.sh` | Genera manifiestos para GitOps |
| `test-endpoints.sh` | Prueba endpoints de ingress |
| `cleanup.sh` | Elimina cluster y limpia cache |

## 🐛 Troubleshooting

### Helmfile command not found

```bash
# Verificar instalación
which helmfile

# Reinstalar si es necesario
# Ver docs/01-setup.md
```

### Pods en CrashLoopBackOff

```bash
# Ver logs
kubectl logs -n dev <pod-name>

# Verificar dependencias
helmfile -f helmfile.d/01-infrastructure.yaml -e dev list

# Ver eventos
kubectl get events -n dev --sort-by='.lastTimestamp'
```

### Chart version not found

```bash
# Error común: version incorrecta en helmfile

# Ver versiones disponibles
helm search repo groundhog2k/postgres --versions

# Actualizar version en helmfile.d/XX-xxx.yaml
version: ~1.5.0  # O la versión disponible
```

### ArgoCD OutOfSync

```bash
# Ver diferencias
argocd app diff helmfile-microservices-dev

# Sync manual
argocd app sync helmfile-microservices-dev
```

Ver [troubleshooting completo](docs/01-setup.md#troubleshooting) en cada capítulo.

## 🎓 Recursos Adicionales

### Helmfile

* [Helmfile Documentation](https://helmfile.readthedocs.io/)
* [Helmfile GitHub](https://github.com/helmfile/helmfile)

### Go Templates

* [Go Template Documentation](https://pkg.go.dev/text/template)
* [Helm Template Guide](https://helm.sh/docs/chart_template_guide/)
* [Sprig Functions](http://masterminds.github.io/sprig/)

### ArgoCD

* [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
* [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

### Mikroways

* [Mikroways](https://mikroways.net/)
* [Repo k8s-base-services](https://gitlab.com/mikroways/k8s/k8s-base-services) (referencia)

## 📄 Licencia

Este proyecto está bajo la Licencia MIT - ver [LICENSE](LICENSE) para detalles.

## 👥 Autores

* **Matias Morawicki** - [@matias-tecnosoul](https://github.com/matias-tecnosoul)
* Basado en mejores prácticas de [Mikroways](https://mikroways.net/)

## 🙏 Agradecimientos

* Inspirado en las mejores prácticas de [Mikroways](https://mikroways.net/)
* Basado en experiencias reales de producción
* Agradecimientos a la comunidad de CNCF

## 🔗 Enlaces Relacionados

### Tutorials Anteriores

* [Tutorial Docker + Helm + GitLab CI + ArgoCD](https://gitlab.com/matias-tecnosoul/tutorial-docker-helm-gitlab-argo)

## 🆘 Ayuda y Soporte

- 📖 [Documentación completa](docs/)
- 🐛 [Reportar issues](https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo/issues)
- 💬 [Discusiones](https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo/discussions)

---

**🚀 Happy Helmfiling!**

> 💡 **Tip**: Usa `helmfile diff` antes de cada `apply`. 
> Es tu mejor amigo para evitar sorpresas en producción.