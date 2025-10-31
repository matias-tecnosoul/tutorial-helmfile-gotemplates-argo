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

## 🏗️ Estructura de Ambientes

```bash
helmfile.d/
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
    └── common.yaml
```

## 📄 Configuración de Ambientes

### helmfile.yaml (actualizado)

```yaml
# helmfile.yaml
---
environments:
  dev:
    kubeContext: kind-helmfile-tutorial
    values:
      - helmfile.d/values/common.yaml
      - helmfile.d/environments/dev/values.yaml
      - helmfile.d/environments/dev/secrets.yaml
  
  staging:
    kubeContext: kind-helmfile-tutorial
    values:
      - helmfile.d/values/common.yaml
      - helmfile.d/environments/staging/values.yaml
      - helmfile.d/environments/staging/secrets.yaml
  
  production:
    kubeContext: kind-helmfile-tutorial  # Cambiar a prod cluster
    values:
      - helmfile.d/values/common.yaml
      - helmfile.d/environments/production/values.yaml
      - helmfile.d/environments/production/secrets.yaml

---
helmfiles:
  - path: helmfile.d/01-infrastructure.yaml
```

### helmfile.d/01-infrastructure.yaml (actualizado)

```yaml
# helmfile.d/01-infrastructure.yaml
---
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
    namespace: {{ .Environment.Name }}
    createNamespace: true
    chart: groundhog2k/postgres
    version: ~0.7.0
    values:
      - values/postgres/values.yaml.gotmpl
    labels:
      tier: infrastructure
      component: database
    condition: postgres.enabled
  
  - name: redis
    namespace: {{ .Environment.Name }}
    createNamespace: true
    chart: groundhog2k/redis
    version: ~0.7.0
    values:
      - values/redis/values.yaml.gotmpl
    labels:
      tier: infrastructure
      component: cache
    condition: redis.enabled
```

## 📦 Values por Ambiente

### helmfile.d/values/common.yaml

```yaml
---
# Configuración base compartida

# Global
baseDomain: example.com
clusterIssuer: letsencrypt-staging

# Feature flags
postgres:
  enabled: true
redis:
  enabled: true

# PostgreSQL base
postgres:
  image:
    repository: postgres
    tag: "15-alpine"
  port: 5432
  databases:
    - appdb
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

# Redis base
redis:
  image:
    repository: redis
    tag: "7-alpine"
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

### helmfile.d/environments/dev/values.yaml

```yaml
---
# Overrides para desarrollo

baseDomain: dev.example.local

# Recursos mínimos
postgres:
  persistence:
    enabled: false
  resources:
    limits:
      cpu: 200m
      memory: 256Mi

redis:
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
```

### helmfile.d/environments/staging/values.yaml

```yaml
---
# Overrides para staging

baseDomain: staging.example.com
clusterIssuer: letsencrypt-staging

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

redis:
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
```

### helmfile.d/environments/production/values.yaml

```yaml
---
# Overrides para producción

baseDomain: example.com
clusterIssuer: letsencrypt-prod

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

redis:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

## 🔐 Secrets (sin SOPS)

### ⚠️ ADVERTENCIA IMPORTANTE

```yaml
# ⚠️ Los secrets en este tutorial están en plain text para simplificar
# En producción REAL:
# 1. NUNCA committees secrets.yaml sin cifrar
# 2. Usa SOPS, Sealed Secrets, o Vault
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
```

### helmfile.d/environments/dev/secrets.yaml.example

```yaml
---
# Template para secrets
# Copia este archivo a secrets.yaml y completa los valores

postgres:
  password: "CHANGE-ME"
  rootPassword: "CHANGE-ME"

redis:
  password: "CHANGE-ME"
```

### helmfile.d/environments/dev/secrets.yaml

```yaml
---
# ⚠️ NO COMMITTEAR ESTE ARCHIVO

postgres:
  password: "dev-postgres-secret-123"
  rootPassword: "dev-root-secret-456"

redis:
  password: "dev-redis-secret-789"
```

### Crear secrets para todos los ambientes

```bash
# Dev
cp helmfile.d/environments/dev/secrets.yaml.example \
   helmfile.d/environments/dev/secrets.yaml

# Staging
cp helmfile.d/environments/staging/secrets.yaml.example \
   helmfile.d/environments/staging/secrets.yaml

# Production
cp helmfile.d/environments/production/secrets.yaml.example \
   helmfile.d/environments/production/secrets.yaml

# Editar cada uno con valores diferentes
# Dev: passwords simples
# Staging: passwords intermedios
# Production: passwords fuertes generados
```

## 🎨 Templates con Secrets

### helmfile.d/values/postgres/values.yaml.gotmpl (actualizado)

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
    value: {{ index .Values.postgres.databases 0 | quote }}
  
  - name: POSTGRES_USER
    value: "appuser"
  
  # Secret desde secrets.yaml
  - name: POSTGRES_PASSWORD
    value: {{ .Values.postgres.password | quote }}
  
  - name: POSTGRES_HOST
    value: postgres.{{ $env }}.svc.cluster.local

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

### helmfile.d/values/redis/values.yaml.gotmpl

```yaml
---
{{ $env := .Environment.Name }}

image:
  repository: {{ .Values.redis.image.repository }}
  tag: {{ .Values.redis.image.tag | quote }}

env:
  - name: REDIS_PASSWORD
    value: {{ .Values.redis.password | quote }}
  
  - name: REDIS_HOST
    value: redis.{{ $env }}.svc.cluster.local

resources:
  {{ .Values.redis.resources | toYaml | nindent 2 }}

labels:
  app: redis
  environment: {{ $env }}
  tier: infrastructure
```

## 🧪 Desplegar por Ambiente

### Desarrollo

```bash
# Ver diferencias
helmfile -e dev diff

# Aplicar
helmfile -e dev apply

# Verificar
kubectl get all -n dev
```

### Staging

```bash
# Aplicar staging
helmfile -e staging apply

# Verificar
kubectl get all -n staging

# Comparar con dev
kubectl get pods -n dev
kubectl get pods -n staging
```

### Production

```bash
# Ver qué se instalará
helmfile -e production diff

# Aplicar (con cuidado)
helmfile -e production apply

# Verificar
kubectl get all -n production
```

## 📊 Comparar Configuraciones

### Ver templates por ambiente

```bash
# Dev
helmfile -e dev template > /tmp/dev.yaml

# Staging
helmfile -e staging template > /tmp/staging.yaml

# Production
helmfile -e production template > /tmp/production.yaml

# Comparar
diff /tmp/dev.yaml /tmp/staging.yaml
diff /tmp/staging.yaml /tmp/production.yaml
```

### Ver solo diferencias de recursos

```bash
# Extraer solo resources de cada ambiente
helmfile -e dev template | yq '.resources' > /tmp/resources-dev.yaml
helmfile -e production template | yq '.resources' > /tmp/resources-prod.yaml

diff /tmp/resources-dev.yaml /tmp/resources-prod.yaml
```

## 🎯 Feature Flags

### Habilitar/deshabilitar componentes

```yaml
# environments/dev/values.yaml
postgres:
  enabled: true
redis:
  enabled: true

# environments/staging/values.yaml
postgres:
  enabled: true
redis:
  enabled: true

# environments/production/values.yaml
postgres:
  enabled: true
redis:
  enabled: true
```

### Aplicar solo componentes habilitados

```bash
# PostgreSQL está habilitado en dev
helmfile -e dev -l component=database apply

# Deshabilitar Redis temporalmente
# Edit: environments/dev/values.yaml
redis:
  enabled: false

# Redis no se desplegará
helmfile -e dev apply
```

## 🔄 Workflow Típico

### 1. Desarrollar en dev

```bash
# Hacer cambios en values/common.yaml
nano helmfile.d/values/common.yaml

# Probar en dev
helmfile -e dev diff
helmfile -e dev apply

# Verificar
kubectl get all -n dev
```

### 2. Promover a staging

```bash
# Aplicar a staging
helmfile -e staging apply

# Verificar diferencias con dev
diff <(helmfile -e dev template) <(helmfile -e staging template)
```

### 3. Desplegar a production

```bash
# Review exhaustivo
helmfile -e production diff

# Aplicar con confirmación
helmfile -e production apply

# Monitorear
kubectl get pods -n production -w
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
helmfile -e production diff

# Luego apply
helmfile -e production apply
```

### 3. Usar naming consistente

```yaml
# Namespaces según ambiente
namespace: {{ .Environment.Name }}

# Recursos con prefijo de ambiente
name: postgres-{{ .Environment.Name }}
```

### 4. Documentar overrides

```yaml
# environments/production/values.yaml
---
# Production overrides
# - Más recursos (4Gi RAM)
# - Persistencia habilitada (20Gi)
# - Backups automáticos

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
helmfile -e dev list

# Ver valores cargados (⚠️ muestra secrets en consola)
helmfile -e dev write-values
```

### Valores no se sobreescriben

```bash
# Orden importa: último gana
environments:
  dev:
    values:
      - common.yaml        # 1. Base
      - dev/values.yaml    # 2. Override
      - dev/secrets.yaml   # 3. Secrets (gana)
```

### Diferencias inesperadas entre ambientes

```bash
# Ver qué valores tiene cada ambiente
helmfile -e dev write-values > /tmp/dev-values.yaml
helmfile -e production write-values > /tmp/prod-values.yaml

diff /tmp/dev-values.yaml /tmp/prod-values.yaml
```

## ✅ Checklist

- [ ] Creaste estructura environments/ con dev/staging/production
- [ ] Configuraste herencia: common → values → secrets
- [ ] Agregaste .gitignore para secrets.yaml
- [ ] Creaste secrets.yaml.example para cada ambiente
- [ ] Desplegaste en dev y staging exitosamente
- [ ] Comparaste configuraciones con diff
- [ ] Validaste que secrets se cargan correctamente

## ➡️ Siguiente Paso

👉 **[05 - Helmfile Modular](05-helmfile-modular.md)**

Aprenderás a:
- Dividir helmfile.yaml en módulos (helmfile.d/)
- Organizar por categorías (infrastructure, services, ingress)
- Pattern de Mikroways para proyectos grandes
- Deploy selectivo por módulo

---

**💡 Tip**: Usa `helmfile -e production diff` antes de cada deploy a producción. Revisa CADA cambio antes de aplicar.