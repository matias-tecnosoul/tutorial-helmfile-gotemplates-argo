# 04 - Multi-Ambiente (45 min)

## 🎯 Objetivo

Gestionar múltiples ambientes (dev/staging/production) con un solo código base, usando herencia de valores y secrets.

## 📝 Patrón de Herencia
```
common.yaml (base)
    ↓
dev/values.yaml (override)
    ↓
dev/secrets.yaml (sensitive)
    ↓
Configuración final
```

## 🔍 Comparación con k8s-base-services

> 💡 **Nota sobre patterns de Mikroways:**
> 
> Este tutorial usa un pattern ligeramente diferente a los k8s-base-services.
>
> | Aspecto | k8s-base-services | Este tutorial |
> |---------|-------------------|---------------|
> | **Ambientes** | 1 (production) | 3 (dev/staging/prod) |
> | **Values** | `environments/production/values.yaml` | `common.yaml` + overrides |
> | **Herencia** | No | Sí (common → env → secrets) |
> | **Uso** | Servicios base de infra | App + multi-env |
>
> **Ambos patterns son válidos** y siguen el principio de helmfiles modulares.
> Elegimos el pattern con herencia porque es más didáctico para aprender
> a gestionar múltiples ambientes con configuración compartida.

## ⚠️ Nota sobre Namespace

> **Namespace hardcoded en este tutorial:**
> 
> Para simplificar, usamos `namespace: dev` hardcoded en los módulos.
> 
> En un proyecto real multi-ambiente, tendrías:
> - Opción 1: Módulos separados por ambiente (`01-infrastructure-dev.yaml`, `01-infrastructure-prod.yaml`)
> - Opción 2: Usar `.yaml.gotmpl` y `{{ .Environment.Name }}`
> 
> Para este tutorial, nos enfocamos en aprender Helmfile con un solo ambiente.

## 🏗️ Estructura de Ambientes
```bash
helmfile.d/
├── 01-infrastructure.yaml       # Módulo (namespace: dev)
├── 02-services.yaml             # Módulo (namespace: dev)
├── 03-ingress.yaml              # Módulo (OPCIONAL)
├── environments/
│   ├── dev/
│   │   ├── values.yaml
│   │   ├── secrets.yaml
│   │   └── secrets.yaml.example
│   ├── staging/
│   │   ├── values.yaml
│   │   ├── secrets.yaml
│   │   └── secrets.yaml.example
│   └── production/
│       ├── values.yaml
│       ├── secrets.yaml
│       └── secrets.yaml.example
└── values/
    ├── common.yaml              # Base compartida
    ├── postgres/
    │   └── values.yaml.gotmpl
    └── app-service/
        └── values.yaml.gotmpl
```

## 📄 Configuración de Ambientes en Módulo

### helmfile.d/01-infrastructure.yaml
```yaml
---
# Definir ambientes en el módulo
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
    namespace: dev  # Hardcoded para este tutorial
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

## 📦 Values por Ambiente

### helmfile.d/values/common.yaml
```yaml
---
# Configuración base compartida

# Global
baseDomain: example.com

# PostgreSQL base
postgres:
  enabled: true
  image:
    repository: postgres
    tag: "15-alpine"
  port: 5432
  persistence:
    enabled: false
    size: 1Gi
    className: standard
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

# App Service base
appService:
  enabled: true
  image:
    repository: nginx  # Placeholder
    tag: alpine
  replicaCount: 1
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  ingress:
    enabled: false
```

### helmfile.d/environments/dev/values.yaml
```yaml
---
# Overrides para desarrollo

baseDomain: dev.example.local

# Recursos mínimos para dev
postgres:
  persistence:
    enabled: false  # Sin persistencia en dev
  resources:
    limits:
      cpu: 200m
      memory: 256Mi

appService:
  ingress:
    enabled: false  # Port-forward recomendado en dev
```

### helmfile.d/environments/staging/values.yaml
```yaml
---
# Overrides para staging

baseDomain: staging.example.com

postgres:
  persistence:
    enabled: true
    size: 5Gi
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

appService:
  replicaCount: 2
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
  ingress:
    enabled: true  # Ingress en staging
```

### helmfile.d/environments/production/values.yaml
```yaml
---
# Overrides para producción

baseDomain: example.com

postgres:
  persistence:
    enabled: true
    size: 20Gi
    className: gp3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

appService:
  replicaCount: 3
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  ingress:
    enabled: true
```

## 🔐 Secrets (sin SOPS)

### ⚠️ ADVERTENCIA IMPORTANTE
```yaml
# ⚠️ Los secrets en este tutorial están en plain text para simplificar
# En producción REAL:
# 1. NUNCA committees secrets.yaml sin cifrar
# 2. Usa SOPS, Sealed Secrets, o External Secrets
# 3. Ver tutorial futuro sobre secrets management
```

### .gitignore
```bash
# .gitignore (raíz del proyecto)

# Secrets (NO committear)
helmfile.d/environments/*/secrets.yaml

# Keep examples
!helmfile.d/environments/*/secrets.yaml.example

# Local
.envrc
.kube/
.helm/

# App build artifacts
app-service-src/node_modules/
app-service-src/npm-debug.log
```

### helmfile.d/environments/dev/secrets.yaml.example
```yaml
---
# Template para secrets
# Copia este archivo a secrets.yaml y completa los valores

postgres:
  password: "CHANGE-ME"
```

### helmfile.d/environments/dev/secrets.yaml
```yaml
---
# ⚠️ NO COMMITTEAR ESTE ARCHIVO

postgres:
  password: "dev-postgres-secret-123"
```

### Crear secrets para todos los ambientes
```bash
# Staging
cp helmfile.d/environments/dev/secrets.yaml.example \
   helmfile.d/environments/staging/secrets.yaml.example

nano helmfile.d/environments/staging/secrets.yaml
# Contenido:
# postgres:
#   password: "staging-stronger-password-456"

# Production
cp helmfile.d/environments/dev/secrets.yaml.example \
   helmfile.d/environments/production/secrets.yaml.example

nano helmfile.d/environments/production/secrets.yaml
# Contenido:
# postgres:
#   password: "VERY-STRONG-PROD-PASSWORD-789"
```

## 🎨 Templates con Secrets

### helmfile.d/values/postgres/values.yaml.gotmpl
```yaml
---
{{ $env := .Environment.Name }}
{{ $isProd := eq $env "production" }}

image:
  repository: {{ .Values.postgres.image.repository }}
  tag: {{ .Values.postgres.image.tag | quote }}

{{ if $isProd }}
replicaCount: 3
{{ else }}
replicaCount: 1
{{ end }}

env:
  - name: POSTGRES_DB
    value: "appdb"
  
  - name: POSTGRES_USER
    value: "appuser"
  
  # Secret desde secrets.yaml
  - name: POSTGRES_PASSWORD
    value: {{ .Values.postgres.password | quote }}

resources:
  {{ .Values.postgres.resources | toYaml | nindent 2 }}

{{ with .Values.postgres.persistence }}
{{ if .enabled }}
storage:
  className: {{ .className }}
  requestedSize: {{ .size }}
{{ end }}
{{ end }}

labels:
  app: postgres
  environment: {{ $env }}
  tier: infrastructure
  {{ if $isProd }}
  critical: "true"
  {{ end }}
```

### helmfile.d/values/app-service/values.yaml.gotmpl
```yaml
---
{{ $env := .Environment.Name }}

replicaCount: {{ .Values.appService.replicaCount }}

image:
  repository: {{ .Values.appService.image.repository }}
  tag: {{ .Values.appService.image.tag }}
  pullPolicy: {{ if eq $env "production" }}IfNotPresent{{ else }}Always{{ end }}

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

env:
  - name: NODE_ENV
    value: {{ $env }}
  
  - name: PORT
    value: "3000"
  
  - name: DB_HOST
    value: postgres.dev.svc.cluster.local
  
  - name: DB_NAME
    value: "appdb"
  
  - name: DB_USER
    value: "appuser"
  
  # Secret compartido con postgres
  - name: DB_PASSWORD
    value: {{ .Values.postgres.password | quote }}

resources:
  {{ .Values.appService.resources | toYaml | nindent 2 }}

{{ if .Values.appService.ingress.enabled }}
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.{{ .Values.baseDomain }}
      paths:
        - path: /
          pathType: Prefix
  {{ if eq $env "production" }}
  tls:
    - secretName: app-service-tls
      hosts:
        - app.{{ .Values.baseDomain }}
  {{ end }}
{{ end }}
```

## 🧪 Desplegar por Ambiente

### Desarrollo
```bash
# Ver diferencias
helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff

# Aplicar infraestructura
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

### Desplegar app-service en dev
```bash
# Aplicar services
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
```

### Probar la aplicación
```bash
# Port-forward a app-service
kubectl port-forward -n dev svc/app-service 3000:80

# En otra terminal, probar endpoints
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Test task from dev"}'
```

**Salida esperada:**
```json
// GET /health
{
  "status": "healthy",
  "db": "connected",
  "version": "1.0.0",
  "timestamp": "2024-01-15T10:30:00.000Z"
}

// GET /api/tasks
[
  {
    "id": 1,
    "title": "Setup Kubernetes cluster",
    "completed": true,
    "created_at": "2024-01-15T10:00:00.000Z"
  },
  {
    "id": 2,
    "title": "Deploy with Helmfile",
    "completed": false,
    "created_at": "2024-01-15T10:00:00.000Z"
  }
]
```

### Comparar configuraciones entre ambientes
```bash
# Ver templates por ambiente
helmfile -f helmfile.d/01-infrastructure.yaml -e dev template > /tmp/dev.yaml
helmfile -f helmfile.d/01-infrastructure.yaml -e staging template > /tmp/staging.yaml
helmfile -f helmfile.d/01-infrastructure.yaml -e production template > /tmp/production.yaml

# Comparar dev vs staging
diff /tmp/dev.yaml /tmp/staging.yaml

# Comparar staging vs production
diff /tmp/staging.yaml /tmp/production.yaml
```

**Diferencias esperadas (dev vs production):**
```diff
# Réplicas
- replicas: 1
+ replicas: 3

# Memoria
- memory: 256Mi
+ memory: 4Gi

# Persistencia
- # Sin storage
+ storage:
+   className: gp3
+   requestedSize: 20Gi

# Labels
  environment: dev
+ critical: "true"
```

## 📊 Ver valores mergeados
```bash
# Ver cómo Helmfile mergea los valores en dev
helmfile -f helmfile.d/01-infrastructure.yaml -e dev write-values

# Guardar para inspección
helmfile -f helmfile.d/01-infrastructure.yaml -e dev write-values > /tmp/merged-dev.yaml

# Ver producción
helmfile -f helmfile.d/01-infrastructure.yaml -e production write-values > /tmp/merged-prod.yaml

# Comparar merge
diff /tmp/merged-dev.yaml /tmp/merged-prod.yaml
```

## 🔄 Workflow Típico

### 1. Desarrollar en dev
```bash
# Hacer cambios en common.yaml o dev/values.yaml
nano helmfile.d/values/common.yaml

# Probar localmente
helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# Verificar
kubectl get all -n dev
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/health
```

### 2. Promover a staging (simulado)
```bash
# Ver qué cambiaría en staging
helmfile -f helmfile.d/01-infrastructure.yaml -e staging diff

# Aplicar a staging (si tuvieras cluster staging)
# helmfile -f helmfile.d/01-infrastructure.yaml -e staging apply

# Verificar diferencias de configuración
diff <(helmfile -f helmfile.d/01-infrastructure.yaml -e dev template) \
     <(helmfile -f helmfile.d/01-infrastructure.yaml -e staging template)
```

### 3. Desplegar a production (simulado)
```bash
# Review exhaustivo
helmfile -f helmfile.d/01-infrastructure.yaml -e production diff

# Ver template completo antes de aplicar
helmfile -f helmfile.d/01-infrastructure.yaml -e production template | less

# Aplicar con cuidado (si tuvieras cluster production)
# helmfile -f helmfile.d/01-infrastructure.yaml -e production apply
```

## 🎯 Feature Flags

### Habilitar/deshabilitar componentes
```yaml
# environments/dev/values.yaml
postgres:
  enabled: true
appService:
  enabled: true

# environments/staging/values.yaml
postgres:
  enabled: true
appService:
  enabled: true

# environments/production/values.yaml
postgres:
  enabled: true
appService:
  enabled: true
```

### Deshabilitar temporalmente un componente
```bash
# Edit: environments/dev/values.yaml
appService:
  enabled: false

# App-service no se desplegará
helmfile -f helmfile.d/02-services.yaml -e dev apply
# Output: No releases to deploy (condition not met)
```

## 🛡️ Best Practices

### 1. Nunca usar secretos hardcodeados
```yaml
# ❌ MAL
env:
  - name: DB_PASSWORD
    value: "my-secret-123"

# ✅ BIEN
env:
  - name: DB_PASSWORD
    value: {{ .Values.postgres.password | quote }}
```

### 2. Validar antes de aplicar
```bash
# Siempre diff primero
helmfile -f helmfile.d/01-infrastructure.yaml -e production diff

# Luego apply
helmfile -f helmfile.d/01-infrastructure.yaml -e production apply
```

### 3. Usar naming consistente
```yaml
# Labels con ambiente
labels:
  app: postgres
  environment: {{ .Environment.Name }}
  tier: infrastructure
```

### 4. Documentar overrides
```yaml
# environments/production/values.yaml
---
# Production overrides
# - Más recursos (4Gi RAM)
# - Persistencia habilitada (20Gi)
# - 3 réplicas para alta disponibilidad

postgres:
  resources:
    limits:
      memory: 4Gi
```

## 🐛 Troubleshooting

### Secrets no se cargan
```bash
# Verificar que secrets.yaml existe
ls -la helmfile.d/environments/dev/secrets.yaml

# Verificar que está en la lista de values
grep -A5 "environment:" helmfile.d/01-infrastructure.yaml

# Ver valores cargados (⚠️ muestra secrets en consola)
helmfile -f helmfile.d/01-infrastructure.yaml -e dev write-values | grep password
```

### Valores no se sobreescriben
```bash
# Orden importa: último gana
# Verificar orden en environments:
#   - values/common.yaml        # 1. Base
#   - environments/dev/values.yaml    # 2. Override
#   - environments/dev/secrets.yaml   # 3. Secrets (gana)
```

### Diferencias inesperadas entre ambientes
```bash
# Ver qué valores tiene cada ambiente
helmfile -f helmfile.d/01-infrastructure.yaml -e dev write-values > /tmp/dev-values.yaml
helmfile -f helmfile.d/01-infrastructure.yaml -e production write-values > /tmp/prod-values.yaml

diff /tmp/dev-values.yaml /tmp/prod-values.yaml
```

### App-service no puede conectar a PostgreSQL
```bash
# Verificar que postgres está corriendo
kubectl get pods -n dev -l app.kubernetes.io/name=postgres

# Verificar service
kubectl get svc -n dev postgres

# Ver logs de app-service
kubectl logs -n dev -l app=app-service

# Probar conexión desde app-service pod
kubectl exec -it -n dev deployment/app-service -- \
  wget -qO- postgres.dev.svc.cluster.local:5432
```

## ✅ Checklist

- [ ] Creaste estructura environments/ con dev/staging/production
- [ ] Configuraste herencia: common → values → secrets
- [ ] Agregaste .gitignore para secrets.yaml
- [ ] Creaste secrets.yaml.example para cada ambiente
- [ ] Desplegaste infraestructura en dev exitosamente
- [ ] Desplegaste app-service en dev exitosamente
- [ ] Probaste endpoints con port-forward
- [ ] Comparaste configuraciones con diff
- [ ] Validaste que secrets se cargan correctamente
- [ ] App-service se conecta a PostgreSQL correctamente

## ➡️ Siguiente Paso

👉 **[05 - Helmfile Modular](05-helmfile-modular.md)**

Aprenderás a:
- Dividir en módulos (helmfile.d/)
- Organizar por categorías (infrastructure, services)
- Pattern de Mikroways para proyectos grandes
- Deploy selectivo por módulo

---

**💡 Tip**: Usa `helmfile -f helmfile.d/01-infrastructure.yaml -e production diff` antes de cada deploy a producción. Revisa CADA cambio antes de aplicar.