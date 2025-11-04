# 05 - Helmfile Modular (45 min)

## 🎯 Objetivo

Entender la organización modular con `helmfile.d/`, facilitando el mantenimiento y deploy selectivo por categoría.

## 📝 ¿Por Qué Modular?

### Problema: Helmfile Monolítico
```yaml
# helmfile.yaml - 200+ líneas
releases:
  - name: postgres      # Líneas 1-30
  - name: app-service   # Líneas 31-60
  - name: ingress-nginx # Líneas 61-90
  # ... más releases
```

**Problemas:**
- Difícil de navegar
- Git conflicts frecuentes
- No puedes desplegar solo infraestructura
- Mezcla concerns (DB, apps, networking)

### Solución: helmfile.d/
```
helmfile.d/
├── 01-infrastructure.yaml  # PostgreSQL
├── 02-services.yaml        # app-service
└── 03-ingress.yaml         # Networking (OPCIONAL)
```

**Ventajas:**
- Separación de responsabilidades
- Deploy selectivo: `helmfile -f helmfile.d/01-infrastructure.yaml apply`
- Menos conflictos en Git
- Fácil onboarding

## 🏗️ Estructura Modular Completa
```bash
tutorial-helmfile-gotemplates-argo/
├── helmfile.d/
│   ├── 01-infrastructure.yaml       # Bases de datos
│   ├── 02-services.yaml             # Aplicaciones
│   ├── 03-ingress.yaml              # Networking (OPCIONAL)
│   ├── environments/                # Por ambiente
│   │   ├── dev/
│   │   │   ├── values.yaml
│   │   │   ├── secrets.yaml
│   │   │   └── secrets.yaml.example
│   │   ├── staging/
│   │   └── production/
│   └── values/                      # Por componente
│       ├── common.yaml
│       ├── postgres/
│       │   └── values.yaml.gotmpl
│       ├── app-service/
│       │   └── values.yaml.gotmpl
│       └── nginx-ingress/           # OPCIONAL
│           └── values.yaml.gotmpl
└── charts/                          # Charts custom
    └── app-service/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

> 💡 **Patrón Mikroways**: 
> Este tutorial NO usa `helmfile.yaml` en la raíz. 
> Cada módulo se ejecuta independientemente, permitiendo deploy selectivo.

## 🗄️ Módulo 1: Infraestructura

### helmfile.d/01-infrastructure.yaml
```yaml
---
# Heredar configuración de ambientes
environments:
  dev:
    kubeContext: kind-helmfile-tutorial
    values:
      - values/common.yaml
      - environments/dev/values.yaml
      - environments/dev/secrets.yaml
  staging:
    kubeContext: kind-helmfile-tutorial
    values:
      - values/common.yaml
      - environments/staging/values.yaml
      - environments/staging/secrets.yaml
  production:
    kubeContext: kind-helmfile-tutorial
    values:
      - values/common.yaml
      - environments/production/values.yaml
      - environments/production/secrets.yaml

---
repositories:
  - name: groundhog2k
    url: https://groundhog2k.github.io/helm-charts/

---
releases:
  - name: postgres
    namespace: dev
    createNamespace: true
    chart: groundhog2k/postgres
    version: ~1.5.0
    values:
      - values/postgres/values.yaml.gotmpl
    wait: true
    timeout: 300
    labels:
      tier: infrastructure
      component: database
    condition: postgres.enabled
```

**Responsabilidad:** Base de datos y servicios de infraestructura base.

## 🎯 Módulo 2: Services

### helmfile.d/02-services.yaml
```yaml
---
environments:
  dev:
    values:
      - values/common.yaml
      - environments/dev/values.yaml
      - environments/dev/secrets.yaml
  staging:
    values:
      - values/common.yaml
      - environments/staging/values.yaml
      - environments/staging/secrets.yaml
  production:
    values:
      - values/common.yaml
      - environments/production/values.yaml
      - environments/production/secrets.yaml

---
releases:
  - name: app-service
    namespace: dev
    chart: ../charts/app-service
    values:
      - values/app-service/values.yaml.gotmpl
    wait: true
    timeout: 300
    needs:
      - dev/postgres
    labels:
      tier: services
      component: app
    condition: appService.enabled
```

**Responsabilidad:** Aplicaciones del negocio que dependen de infraestructura.

**Nota sobre `needs:`** - Esto lo veremos en detalle en el capítulo 06.

## 🌐 Módulo 3: Ingress (OPCIONAL)

### helmfile.d/03-ingress.yaml
```yaml
# ⚠️ OPCIONAL: Este módulo es opcional. Ver docs/07-ingress.md
# Para testing rápido, usa: kubectl port-forward -n dev svc/app-service 3000:80
---
environments:
  dev:
    values:
      - values/common.yaml
      - environments/dev/values.yaml
  staging:
    values:
      - values/common.yaml
      - environments/staging/values.yaml
  production:
    values:
      - values/common.yaml
      - environments/production/values.yaml

---
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx

---
releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    createNamespace: true
    chart: ingress-nginx/ingress-nginx
    version: ~4.11.0
    values:
      - values/nginx-ingress/values.yaml.gotmpl
    wait: true
    timeout: 300
    labels:
      tier: networking
      component: ingress
    condition: ingressNginx.enabled
    needs:
      - dev/app-service
```

**Responsabilidad:** Exponer aplicaciones al exterior (opcional).

## 🎮 Comandos Modulares

### Listar por módulo
```bash
# Toda la infraestructura
helmfile -f helmfile.d/01-infrastructure.yaml -e dev list

# Todos los servicios
helmfile -f helmfile.d/02-services.yaml -e dev list

# Solo ingress (opcional)
helmfile -f helmfile.d/03-ingress.yaml -e dev list
```

**Salida esperada:**
```
# 01-infrastructure.yaml
NAME     NAMESPACE  ENABLED  LABELS                                  CHART
postgres dev        true     component:database,tier:infrastructure  groundhog2k/postgres

# 02-services.yaml
NAME         NAMESPACE  ENABLED  LABELS                      CHART
app-service  dev        true     component:app,tier:services ../charts/app-service
```

### Deploy selectivo
```bash
# Solo infraestructura
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# Solo servicios (requiere infraestructura ya desplegada)
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Solo ingress (opcional)
helmfile -f helmfile.d/03-ingress.yaml -e dev apply
```

### Deploy por labels
```bash
# Solo databases
helmfile -f helmfile.d/01-infrastructure.yaml -e dev -l component=database apply

# Toda la infraestructura
helmfile -f helmfile.d/01-infrastructure.yaml -e dev -l tier=infrastructure apply

# Todos los servicios
helmfile -f helmfile.d/02-services.yaml -e dev -l tier=services apply

# Solo app
helmfile -f helmfile.d/02-services.yaml -e dev -l component=app apply
```

## 📊 Flujo de Deploy
```
Opción 1: Deploy módulo por módulo
    ↓
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
    ↓
  postgres (deployed)
    ↓
helmfile -f helmfile.d/02-services.yaml -e dev apply
    ↓
  app-service (deployed, needs: postgres)
    ↓
helmfile -f helmfile.d/03-ingress.yaml -e dev apply (OPCIONAL)
    ↓
  ingress-nginx (deployed, needs: app-service)
```

## 🧪 Verificar Deploy Modular

### 1. Infraestructura
```bash
# Deploy
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# Verificar
kubectl get all -n dev
```

**Salida esperada:**
```
NAME             READY   STATUS    RESTARTS   AGE
pod/postgres-0   1/1     Running   0          1m

NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/postgres   ClusterIP   10.96.100.50   <none>        5432/TCP   1m

NAME                        READY   AGE
statefulset.apps/postgres   1/1     1m
```

### 2. Services
```bash
# Deploy
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Verificar
kubectl get all -n dev
```

**Salida esperada:**
```
NAME                               READY   STATUS    RESTARTS   AGE
pod/postgres-0                     1/1     Running   0          2m
pod/app-service-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/postgres      ClusterIP   10.96.100.50    <none>        5432/TCP   2m
service/app-service   ClusterIP   10.96.200.100   <none>        80/TCP     30s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-service   1/1     1            1           30s

NAME                        READY   AGE
statefulset.apps/postgres   1/1     2m
```

### 3. Probar la aplicación
```bash
# Port-forward
kubectl port-forward -n dev svc/app-service 3000:80

# En otra terminal
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
```

**Salida esperada:**
```json
{
  "status": "healthy",
  "db": "connected",
  "version": "1.0.0"
}

[
  {
    "id": 1,
    "title": "Setup Kubernetes cluster",
    "completed": true
  },
  {
    "id": 2,
    "title": "Deploy with Helmfile",
    "completed": false
  }
]
```

## 🎯 Patrones de Organización

### Por tipo de recurso (Mikroways) ✅ USAMOS ESTE
```
helmfile.d/
├── 01-infrastructure.yaml    # DB, cache
├── 02-services.yaml          # Apps
├── 03-ingress.yaml           # Networking
```

**Ventajas:**
- Deploy por capa (infra → apps → networking)
- Dependencias claras entre capas
- Usado en producción real (Mikroways)

### Por dominio de negocio (alternativa)
```
helmfile.d/
├── 01-shared.yaml            # Infra compartida
├── 02-user-domain.yaml       # User + related
├── 03-task-domain.yaml       # Tasks + related
```

**Ventajas:**
- Equipos separados por dominio
- Deploy por feature/dominio

### Por criticidad (alternativa)
```
helmfile.d/
├── 01-critical.yaml          # Core services
├── 02-standard.yaml          # Normal priority
├── 03-optional.yaml          # Nice to have
```

**Ventajas:**
- Deploy priorizando lo crítico
- Rollback selectivo

## 📝 Convenciones de Numeración
```
01-  Base layer (databases, cache)
02-  Application layer (business logic)
03-  Presentation layer (ingress, API gw)
04-  Observability (monitoring, logging)
05-  Security (policies, scanners)
```

> 💡 **Tip**: La numeración ayuda a ver el orden de dependencia de un vistazo.

## 🔄 Ventajas vs Desventajas

### ✅ Ventajas

- **Separación clara** - Cada módulo tiene una responsabilidad
- **Deploy selectivo** - Solo infra, solo apps, etc.
- **Menos conflictos** - Equipos trabajan en módulos diferentes
- **Fácil onboarding** - Nuevo dev solo ve lo relevante
- **Escalable** - Funciona con 5 o 50 releases

### ⚠️ Desventajas

- **Más archivos** - 3 archivos en vez de 1
- **Paths relativos** - `../charts/` puede confundir
- **Overhead** - Para proyectos muy pequeños (<3 releases)
- **Duplicación** - Configuración de ambientes en cada módulo

> 💡 **Cuándo usar módulos:**
> - ✅ Proyectos con 5+ releases
> - ✅ Equipos múltiples
> - ✅ Deploy selectivo necesario
> - ❌ Proyecto muy simple (2-3 releases)

## 🐛 Troubleshooting

### Paths relativos incorrectos
```yaml
# ❌ ERROR (desde helmfile.d/02-services.yaml)
chart: charts/app-service  # No encuentra el chart

# ✅ CORRECTO
chart: ../charts/app-service  # Path relativo al helmfile
```

### Ambientes no heredados

Cada módulo debe declarar sus propios `environments:`:
```yaml
# helmfile.d/02-services.yaml
environments:
  dev:
    values:
      - values/common.yaml       # ✅ Correcto
      - environments/dev/values.yaml
      - environments/dev/secrets.yaml
```

Helmfile **no hereda automáticamente** environments del módulo anterior.

### Dependencies entre módulos
```yaml
# ❌ No funciona (postgres en otro módulo)
needs:
  - postgres

# ✅ Correcto (incluir namespace)
needs:
  - dev/postgres
```

### Deploy en orden incorrecto
```bash
# ❌ ERROR: Deploy services antes de infra
helmfile -f helmfile.d/02-services.yaml -e dev apply
# Error: app-service needs postgres (no existe aún)

# ✅ CORRECTO: Deploy en orden
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply  # Primero infra
helmfile -f helmfile.d/02-services.yaml -e dev apply        # Luego services
```

## 🎓 Ejercicio Práctico

**Objetivo:** Actualizar solo la infraestructura sin tocar services.
```bash
# 1. Cambiar recursos de postgres
nano helmfile.d/values/common.yaml
# Aumentar memory: 1Gi

# 2. Ver diferencias solo en infra
helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff

# 3. Aplicar solo infra
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# 4. Verificar que services NO se tocó
helmfile -f helmfile.d/02-services.yaml -e dev diff
# Output: No changes
```

## ✅ Checklist

- [ ] Entiendes por qué usar módulos
- [ ] Tienes 3 módulos: 01-infrastructure, 02-services, 03-ingress
- [ ] Cada módulo declara sus environments
- [ ] Puedes hacer deploy selectivo por módulo
- [ ] Puedes hacer deploy selectivo por labels
- [ ] Entiendes paths relativos (`../charts/`)
- [ ] Deploy en orden correcto funciona (infra → services)
- [ ] Entiendes el patrón de Mikroways

## ➡️ Siguiente Paso

👉 **[06 - Dependencias](06-dependencies.md)**

Aprenderás:
- Dependencias con `needs:`
- Orden de ejecución
- Wait conditions
- Dependencias entre módulos

---

**💡 Tip**: Para proyectos pequeños (<5 releases), un solo helmfile.yaml es suficiente. 
Usa helmfile.d/ cuando el proyecto crece o trabajas en equipo.