# Tutorial Helmfile + Go Templates

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Tutorial completo y práctico para gestionar aplicaciones en Kubernetes usando Helmfile y Go Templates, siguiendo el patrón modular de Mikroways.

## 🎯 Objetivos de Aprendizaje

Al completar este tutorial, aprenderás a:

- ✅ Gestionar aplicaciones con **Helmfile**
- ✅ Crear configuraciones dinámicas con **Go Templates**
- ✅ Organizar proyectos con **helmfile.d/** (patrón modular)
- ✅ Gestionar múltiples ambientes (dev/staging/production)
- ✅ Manejar dependencias entre servicios
- ✅ [BONUS] Exponer aplicaciones con **Nginx Ingress**

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
| Direnv      | 2.32+   | [direnv.net](https://direnv.net/docs/installation.html) (opcional) |

## 🚀 Quick Start

### Deploy local en 10 minutos
```bash
# 1. Clonar el repositorio
git clone https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo.git
cd tutorial-helmfile-gotemplates-argo

# 2. Configurar direnv (opcional)
cp .envrc-sample .envrc
direnv allow

# 3. Crear cluster Kind
kind create cluster --config kind-config.yaml

# 4. Configurar secrets
cp helmfile.d/environments/dev/secrets.yaml.example \
   helmfile.d/environments/dev/secrets.yaml

# 5. Desplegar infraestructura
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# 6. Desplegar aplicación
helmfile -f helmfile.d/02-services.yaml -e dev apply

# 7. Probar la aplicación
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
```

## 🏗️ Stack Desplegado
```
┌─────────────────┐
│   app-service   │  ← API REST en Node.js
│  (port 3000)    │     - GET /health
└────────┬────────┘     - GET /api/tasks
         │              - POST /api/tasks
         ↓              - DELETE /api/tasks/:id
┌─────────────────┐
│   PostgreSQL    │  ← Base de datos
│  (groundhog2k)  │
└─────────────────┘
```

**Componentes:**
- **PostgreSQL** - Base de datos (Helm chart: groundhog2k/postgres)
- **app-service** - API REST simple en Node.js + Express + pg
- **[OPCIONAL]** Nginx Ingress Controller

**Sin:**
- ❌ Redis, auth-service, user-service, api-gateway (simplificado para aprender Helmfile)
- ❌ ArgoCD (será tutorial separado sobre GitOps)

## 🗂️ Estructura del Proyecto
```
tutorial-helmfile-gotemplates-argo/
├── README.md
├── helmfile.d/                      # Helmfiles modulares (patrón Mikroways)
│   ├── 01-infrastructure.yaml       # PostgreSQL
│   ├── 02-services.yaml             # app-service
│   ├── 03-ingress.yaml              # [OPCIONAL] Nginx Ingress
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
│       ├── app-service/
│       │   └── values.yaml.gotmpl
│       └── nginx-ingress/           # OPCIONAL
│           └── values.yaml.gotmpl
├── charts/                          # Charts custom
│   └── app-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml         # OPCIONAL
├── app-service-src/                 # Código fuente de la app
│   ├── Dockerfile
│   ├── package.json
│   ├── app.js
│   └── README.md
├── scripts/
│   ├── setup-cluster.sh
│   └── cleanup.sh
├── .gitignore
├── .envrc-sample
├── kind-config.yaml
└── docs/                            # Documentación paso a paso
    ├── 01-setup.md
    ├── 02-intro-helmfile.md
    ├── 03-go-templates.md
    ├── 04-multi-env.md
    ├── 05-helmfile-modular.md
    ├── 06-dependencies.md
    └── 07-ingress.md                # OPCIONAL
```

> 💡 **Patrón Mikroways**: Este tutorial NO usa `helmfile.yaml` en la raíz. 
> Cada módulo en `helmfile.d/` se ejecuta independientemente, permitiendo deploy selectivo.

## 📚 Documentación

### Guías Paso a Paso

| # | Capítulo | Tiempo | Descripción |
|---|----------|--------|-------------|
| 1 | [**Setup del Entorno**](docs/01-setup.md) | 30 min | Instalación de herramientas y cluster Kind |
| 2 | [**Introducción a Helmfile**](docs/02-intro-helmfile.md) | 30 min | Qué es Helmfile y deploy de PostgreSQL |
| 3 | [**Go Templates**](docs/03-go-templates.md) | 1h | Variables, condicionales, loops, pipelines |
| 4 | [**Multi-Ambiente**](docs/04-multi-env.md) | 45 min | dev/staging/production y secrets |
| 5 | [**Helmfile Modular**](docs/05-helmfile-modular.md) | 45 min | Patrón helmfile.d/ y deploy selectivo |
| 6 | [**Dependencias**](docs/06-dependencies.md) | 30 min | needs, wait, timeout |
| 7 | [**[BONUS] Ingress**](docs/07-ingress.md) | 30 min | ⚠️ Opcional - Nginx Ingress Controller |

**Tiempo total: ~3h30 (core) + 30 min (bonus opcional)**

### Flujo de Aprendizaje
```
01. Setup
    ↓
02. Helmfile Básico (PostgreSQL)
    ↓
03. Go Templates (dinamismo)
    ↓
04. Multi-Ambiente (dev/staging/prod)
    ↓
05. Modular (helmfile.d/)
    ↓
06. Dependencias (needs)
    ↓
07. [OPCIONAL] Ingress
```

## 🎮 Comandos Útiles
```bash
# Alias útil (opcional)
alias hf-infra='helmfile -f helmfile.d/01-infrastructure.yaml'
alias hf-services='helmfile -f helmfile.d/02-services.yaml'

# Helmfile - Infraestructura
hf-infra -e dev list              # Listar releases
hf-infra -e dev diff              # Ver diferencias
hf-infra -e dev apply             # Aplicar cambios
hf-infra -e dev destroy           # Eliminar todo
hf-infra -e dev template          # Ver manifiestos generados
hf-infra -e dev write-values      # Ver valores mergeados

# Deploy selectivo por labels
hf-infra -e dev -l component=database apply
hf-infra -e dev -l tier=infrastructure apply

# Kubernetes
kubectl get all -n dev
kubectl logs -n dev -l app.kubernetes.io/name=postgres -f
kubectl logs -n dev -l app=app-service -f

# Testing
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Nueva tarea"}'
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

### Probar la aplicación
```bash
# Port-forward
kubectl port-forward -n dev svc/app-service 3000:80

# Health check
curl http://localhost:3000/health

# Listar tareas
curl http://localhost:3000/api/tasks

# Crear tarea
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Aprender Helmfile"}'

# Eliminar tarea
curl -X DELETE http://localhost:3000/api/tasks/1
```

### Verificar deployment
```bash
# Ver todos los recursos
kubectl get all -n dev

# Ver logs de PostgreSQL
kubectl logs -n dev -l app.kubernetes.io/name=postgres -f

# Ver logs de app-service
kubectl logs -n dev -l app=app-service -f

# Describir pod
kubectl describe pod -n dev <pod-name>
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

### 2. Verificar cambios
```bash
# Ver diferencias
kubectl get all -n dev

# Port-forward y probar
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/health
```

### 3. Commit
```bash
git add helmfile.d/
git commit -m "feat: aumentar recursos de postgres"
git push origin main
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

### App-service no conecta a PostgreSQL
```bash
# Verificar que postgres está corriendo
kubectl get pods -n dev -l app.kubernetes.io/name=postgres

# Ver logs de app-service
kubectl logs -n dev -l app=app-service

# Verificar service
kubectl get svc -n dev postgres

# Verificar DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup postgres.dev.svc.cluster.local
```

### Chart version not found
```bash
# Ver versiones disponibles
helm search repo groundhog2k/postgres --versions

# Actualizar version en helmfile.d/01-infrastructure.yaml
version: ~1.5.0  # O la versión disponible
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

### Mikroways

* [Mikroways](https://mikroways.net/)
* [Repo k8s-base-services](https://gitlab.com/mikroways/k8s/k8s-base-services) (referencia)

## 🚀 Próximos Tutoriales

Has completado "Helmfile + Go Templates". Continúa tu aprendizaje con:

### 1. GitOps con Helmfile + ArgoCD
- Despliegue automático desde Git
- Drift detection y auto-heal
- Rollbacks y historial
- Multi-cluster management

**[Próximamente - Tutorial separado]**

### 2. Secrets Management Avanzado
- SOPS + Age encryption
- Sealed Secrets
- External Secrets Operator
- Best practices de seguridad

### 3. Observability Stack
- Prometheus + Grafana
- Loki para logs
- Integración con Helmfile

### 4. Microservices Completos
- Múltiples servicios
- Service mesh (Istio/Linkerd)
- Tracing distribuido

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

### Tutorials Relacionados

* [Tutorial Docker + Helm + GitLab CI + ArgoCD](https://gitlab.com/matias-tecnosoul/tutorial-docker-helm-gitlab-argo) (anterior)

## 🆘 Ayuda y Soporte

- 📖 [Documentación completa](docs/)
- 🐛 [Reportar issues](https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo/issues)
- 💬 [Discusiones](https://github.com/matias-tecnosoul/tutorial-helmfile-gotemplates-argo/discussions)

## 🌟 Por qué este tutorial es diferente

### Simplificado para aprender

En lugar de un stack complejo con 7+ componentes (Redis, auth-service, user-service, api-gateway, etc.), 
este tutorial se enfoca en **2 componentes** para que aprendas Helmfile sin distraerte con arquitectura de microservices.

### Patrón de producción real

Sigue el **patrón modular de Mikroways** (`helmfile.d/` sin `helmfile.yaml` en raíz), 
usado en proyectos reales de producción.

### Progresivo y práctico

Cada capítulo construye sobre el anterior, con ejemplos funcionales que puedes probar inmediatamente.

### Opcional vs Requerido

Claramente marca qué es esencial (caps 1-6) y qué es bonus (cap 7 Ingress), 
permitiéndote elegir tu camino de aprendizaje.

---

**🚀 Happy Helmfiling!**

> 💡 **Tip**: Usa `helmfile diff` antes de cada `apply`. 
> Es tu mejor amigo para evitar sorpresas en producción.