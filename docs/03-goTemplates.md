# 03 - Go Templates (1h)

## 🎯 Objetivo

Dominar Go Templates para crear configuraciones dinámicas y reutilizables, eliminando duplicación de código.

## 📝 ¿Qué son Go Templates?

Go Templates es el lenguaje de templating usado por Helm y Helmfile para generar configuraciones dinámicas.

**Sintaxis básica:**
```yaml
{{ .Values.something }}        # Acceder a valores
{{ if condition }}...{{ end }} # Condicional
{{ range .items }}...{{ end }} # Loop
```

## 🔄 Flujo de carga de valores

Antes de profundizar en templates, es importante entender cómo Helmfile carga y mergea valores:
```
helmfile.d/values/common.yaml (base)
           ↓
helmfile.d/environments/dev/values.yaml (overrides)
           ↓
helmfile.d/environments/dev/secrets.yaml (secrets)
           ↓
    Valores mergeados en .Values
           ↓
helmfile.d/values/postgres/values.yaml.gotmpl
    accede a {{ .Values.* }}
```
> 💡 **Nota sobre `.gotmpl`**: 
> 
> Solo los archivos en `values/servicio/` que usan templates (`{{ }}`) necesitan `.gotmpl`:
> - ✅ `values/postgres/values.yaml.gotmpl` - Usa templates
> - ❌ `values/common.yaml` - Solo valores estáticos (sin .gotmpl)
> - ❌ `environments/dev/secrets.yaml` - Solo valores estáticos (sin .gotmpl)

**Ejemplo práctico:**
```yaml
# 1. common.yaml (base)
postgres:
  image:
    tag: "15-alpine"
  resources:
    limits:
      memory: 512Mi

# 2. environments/dev/values.yaml (override)
postgres:
  resources:
    limits:
      memory: 256Mi  # ← Sobrescribe

# 3. environments/dev/secrets.yaml
postgres:
  password: "dev-postgres-secret-123"

# 4. Resultado final en .Values.postgres:
image:
  tag: "15-alpine"      # De common.yaml
resources:
  limits:
    memory: 256Mi       # De dev/values.yaml (ganó)
password: "dev-postgres-secret-123"  # De dev/secrets.yaml
```

> 💡 **Orden importa**: El último archivo gana en conflictos.

## 🎨 Variables

### Acceso a valores desde .Values
```yaml
# helmfile.d/values/common.yaml
postgres:
  image:
    repository: postgres
    tag: "15-alpine"
  port: 5432

# helmfile.d/environments/dev/secrets.yaml
postgres:
  password: "dev-postgres-secret-123"

# helmfile.d/values/postgres/values.yaml.gotmpl
image:
  repository: {{ .Values.postgres.image.repository }}
  tag: {{ .Values.postgres.image.tag | quote }}

env:
  - name: POSTGRES_PORT
    value: {{ .Values.postgres.port | quote }}
  
  - name: POSTGRES_PASSWORD
    value: {{ .Values.postgres.password | quote }}  # ← De secrets.yaml
```

### Variables locales
```yaml
# Definir variable
{{ $env := .Environment.Name }}
{{ $dbName := "appdb" }}
{{ $port := 5432 }}

# Usar variable
database: {{ $dbName }}
port: {{ $port }}
namespace: {{ $env }}
```

### Contexto especial en Helmfile
```yaml
{{ .Environment.Name }}        # dev, staging, production
{{ .Release.Name }}            # Nombre del release
{{ .Release.Namespace }}       # Namespace del release
{{ .Values.* }}                # Valores mergeados de common + env + secrets
```

## 🔀 Condicionales

### If básico
```yaml
# helmfile.d/values/postgres/values.yaml.gotmpl
---
{{ if eq .Environment.Name "production" }}
# Configuración de producción
replicaCount: 3
resources:
  limits:
    cpu: 2000m
    memory: 4Gi
{{ else }}
# Configuración de desarrollo
replicaCount: 1
resources:
  limits:
    cpu: 500m
    memory: 512Mi
{{ end }}
```

### If/Else If/Else
```yaml
{{ if eq .Environment.Name "production" }}
replicaCount: 3
{{ else if eq .Environment.Name "staging" }}
replicaCount: 2
{{ else }}
replicaCount: 1
{{ end }}
```

### Operadores de comparación
```yaml
{{ if eq .Environment.Name "dev" }}         # Igual
{{ if ne .Environment.Name "dev" }}         # No igual
{{ if and (condition1) (condition2) }}      # AND
{{ if or (condition1) (condition2) }}       # OR
{{ if not .Values.disabled }}               # NOT
{{ if gt .Values.replicas 3 }}              # Greater than
{{ if lt .Values.replicas 3 }}              # Less than
```

### Habilitar/deshabilitar secciones
```yaml
{{ if .Values.postgres.persistence.enabled }}
storage:
  className: {{ .Values.postgres.persistence.className }}
  requestedSize: {{ .Values.postgres.persistence.size }}
{{ end }}

{{ if .Values.postgres.backup.enabled }}
backup:
  schedule: "0 2 * * *"
  retention: 7
{{ end }}
```

## 🔧 Pipelines y Funciones

### Pipelines básicos
```yaml
# quote: Agregar comillas
image: {{ .Values.image.repository | quote }}

# upper/lower: Mayúsculas/minúsculas
env_name: {{ .Environment.Name | upper }}

# default: Valor por defecto si está vacío
tag: {{ .Values.image.tag | default "latest" }}

# trim: Eliminar espacios
name: {{ .Values.name | trim }}
```

### Funciones de formato
```yaml
# toYaml: Convertir a YAML (útil para objetos complejos)
resources:
  {{ .Values.resources | toYaml | nindent 2 }}

# nindent: Indentar N espacios
labels:
  {{ .Values.labels | toYaml | nindent 2 }}

# indent: Similar a nindent
annotations:
  {{ .Values.annotations | toYaml | indent 2 }}
```

### Encadenamiento de funciones
```yaml
# Múltiples pipes
name: {{ .Values.serviceName | trim | lower | quote }}

# Con default y quote
tag: {{ .Values.image.tag | default "latest" | quote }}
```

## 🎯 With (Scope)

### With básico
```yaml
# Sin with (repetitivo)
database:
  host: {{ .Values.postgres.host }}
  port: {{ .Values.postgres.port }}
  name: {{ .Values.postgres.database }}
  user: {{ .Values.postgres.user }}

# Con with (limpio)
{{ with .Values.postgres }}
database:
  host: {{ .host }}
  port: {{ .port }}
  name: {{ .database }}
  user: {{ .user }}
{{ end }}
```

### With con condicional
```yaml
{{ with .Values.postgres.backup }}
{{ if .enabled }}
backup:
  schedule: {{ .schedule }}
  retention: {{ .retention }}
{{ end }}
{{ end }}
```

## 🏗️ Ejemplo Completo: PostgreSQL

### helmfile.d/values/common.yaml
```yaml
---
# Configuración base

baseDomain: example.com

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
  backup:
    enabled: false
    schedule: "0 2 * * *"
    retention: 7
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

### helmfile.d/environments/dev/values.yaml
```yaml
---
# Overrides para dev

baseDomain: dev.example.local

postgres:
  persistence:
    enabled: false  # Sin persistencia en dev
  backup:
    enabled: false  # Sin backup en dev
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
```

### helmfile.d/environments/dev/secrets.yaml
```yaml
---
# Secrets para dev

postgres:
  password: "dev-postgres-secret-123"
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
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retention: 30
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi
```

### helmfile.d/environments/production/secrets.yaml
```yaml
---
# Secrets para producción

postgres:
  password: "STRONG-PROD-PASSWORD-HERE"
```

### helmfile.d/values/postgres/values.yaml.gotmpl
```yaml
---
# Template dinámico de PostgreSQL

{{ $env := .Environment.Name }}
{{ $isProd := eq $env "production" }}

# Imagen
image:
  repository: {{ .Values.postgres.image.repository }}
  tag: {{ .Values.postgres.image.tag | quote }}

# Réplicas según ambiente
{{ if $isProd }}
replicaCount: 3
{{ else }}
replicaCount: 1
{{ end }}

# Variables de entorno
env:
  - name: POSTGRES_DB
    value: "appdb"
  
  - name: POSTGRES_USER
    value: "appuser"
  
  - name: POSTGRES_PASSWORD
    value: {{ .Values.postgres.password | quote }}
  
  - name: POSTGRES_PORT
    value: {{ .Values.postgres.port | quote }}

# Recursos (mergeados de common + env)
resources:
  {{ .Values.postgres.resources | toYaml | nindent 2 }}

# Persistencia (solo si está habilitada)
{{ with .Values.postgres.persistence }}
{{ if .enabled }}
storage:
  className: {{ .className }}
  requestedSize: {{ .size }}
{{ end }}
{{ end }}

# Backup (solo en producción)
{{ with .Values.postgres.backup }}
{{ if and .enabled $isProd }}
backup:
  enabled: true
  schedule: {{ .schedule | quote }}
  retention: {{ .retention }}
  s3:
    bucket: postgres-backups-{{ $env }}
    region: us-east-1
{{ end }}
{{ end }}

# Labels dinámicos
labels:
  app: postgres
  environment: {{ $env }}
  tier: infrastructure
  {{ if $isProd }}
  critical: "true"
  {{ end }}
```

## 🏗️ Ejemplo Completo: App Service

Ahora veamos cómo usar templates para nuestra aplicación Node.js.

### helmfile.d/values/common.yaml (agregar)
```yaml
# App Service
appService:
  enabled: true
  image:
    repository: nginx  # Placeholder - cambiar al buildear imagen
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

### helmfile.d/environments/production/values.yaml (agregar)
```yaml
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

### helmfile.d/values/app-service/values.yaml.gotmpl
```yaml
---
{{ $env := .Environment.Name }}

# Réplicas
replicaCount: {{ .Values.appService.replicaCount }}

# Imagen
image:
  repository: {{ .Values.appService.image.repository }}
  tag: {{ .Values.appService.image.tag }}
  pullPolicy: {{ if eq $env "production" }}IfNotPresent{{ else }}Always{{ end }}

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 3000

# Variables de entorno
env:
  - name: NODE_ENV
    value: {{ $env }}
  
  - name: PORT
    value: "3000"
  
  # Conexión a PostgreSQL
  - name: DB_HOST
    value: postgres.dev.svc.cluster.local
  
  - name: DB_NAME
    value: "appdb"
  
  - name: DB_USER
    value: "appuser"
  
  - name: DB_PASSWORD
    value: {{ .Values.postgres.password | quote }}

# Recursos
resources:
  {{ .Values.appService.resources | toYaml | nindent 2 }}

# Ingress (solo si está habilitado)
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

## 🧪 Probar Templates

### Ver template renderizado
```bash
# Ver PostgreSQL renderizado para dev
helmfile -f helmfile.d/01-infrastructure.yaml -e dev template

# Ver PostgreSQL para production
helmfile -f helmfile.d/01-infrastructure.yaml -e production template

# Ver app-service renderizado (cuando lo despleguemos)
helmfile -f helmfile.d/02-services.yaml -e dev template
```

### Ver valores mergeados
```bash
# Ver cómo Helmfile mergea los valores
helmfile -f helmfile.d/01-infrastructure.yaml -e dev write-values

# Salvar a archivo para inspección
helmfile -f helmfile.d/01-infrastructure.yaml -e dev write-values > /tmp/values-dev.yaml
cat /tmp/values-dev.yaml
```

### Comparar entre ambientes
```bash
# Dev
helmfile -f helmfile.d/01-infrastructure.yaml -e dev template > /tmp/postgres-dev.yaml

# Production
helmfile -f helmfile.d/01-infrastructure.yaml -e production template > /tmp/postgres-prod.yaml

# Comparar
diff /tmp/postgres-dev.yaml /tmp/postgres-prod.yaml
```

**Diferencias esperadas:**
```diff
- replicaCount: 1
+ replicaCount: 3

- memory: 256Mi
+ memory: 4Gi

+ storage:
+   className: gp3
+   requestedSize: 20Gi

+ backup:
+   enabled: true
```

## 🐛 Debugging

### Debug con --debug
```bash
helmfile -f helmfile.d/01-infrastructure.yaml -e dev --debug template
```

Muestra paso a paso la evaluación de templates.

### Comentarios en templates
```yaml
{{/* Esto es un comentario, no aparece en el output */}}

{{/* 
  Comentario
  multilínea
*/}}

# Esto es un comentario YAML (aparece en el output)
```

### Imprimir variables para debug
```yaml
{{/* Debug: Imprimir valor */}}
# DEBUG: Environment = {{ .Environment.Name }}
# DEBUG: IsProd = {{ eq .Environment.Name "production" }}
# DEBUG: Password length = {{ len .Values.postgres.password }}
```

## ⚠️ Errores Comunes

### Error 1: Variable no definida
```yaml
# ❌ ERROR
tag: {{ .Values.image.tag }}

# ✅ CORRECTO (con default)
tag: {{ .Values.image.tag | default "latest" }}
```

### Error 2: Scope en with
```yaml
{{ with .Values.postgres }}
  # Aquí '.' es .Values.postgres, NO .Values
  port: {{ .port }}  # ✅ Correcto
  
  # Para acceder a .Values necesitas $
  domain: {{ $.Values.baseDomain }}  # ✅ Correcto
{{ end }}
```

### Error 3: Indentación incorrecta
```yaml
# ❌ ERROR (mal indentado)
resources:
{{ .Values.resources | toYaml }}

# ✅ CORRECTO
resources:
  {{ .Values.resources | toYaml | nindent 2 }}
```

### Error 4: Quotes en números
```yaml
# ❌ ERROR (puerto como string cuando debe ser número en YAML)
port: {{ .Values.port | quote }}  # "5432"

# ✅ CORRECTO
port: {{ .Values.port }}  # 5432

# ✅ CORRECTO (cuando debe ser string, ej: env var)
env:
  - name: PORT
    value: {{ .Values.port | quote }}  # "5432" (string)
```

### Error 5: Valor no existe en secrets
```yaml
# ❌ ERROR (si password no está en secrets.yaml)
value: {{ .Values.postgres.password }}  # Error: nil pointer

# ✅ CORRECTO (con default)
value: {{ .Values.postgres.password | default "changeme" }}
```

## 📚 Funciones Útiles
```yaml
# Strings
{{ upper "hello" }}              # HELLO
{{ lower "HELLO" }}              # hello
{{ trim " hello " }}             # hello
{{ quote "hello" }}              # "hello"
{{ replace "-" "_" "hello-world" }}  # hello_world

# Números
{{ add 1 2 }}                    # 3
{{ sub 5 2 }}                    # 3
{{ mul 3 4 }}                    # 12
{{ div 10 2 }}                   # 5

# Lógica
{{ and true false }}             # false
{{ or true false }}              # true
{{ not false }}                  # true

# Comparación
{{ eq "a" "a" }}                 # true
{{ ne "a" "b" }}                 # true
{{ lt 1 2 }}                     # true
{{ gt 2 1 }}                     # true

# Conversión
{{ toYaml .Values }}             # Objeto a YAML
{{ toJson .Values }}             # Objeto a JSON
{{ toString 123 }}               # "123"

# Default
{{ .Values.tag | default "latest" }}

# Longitud
{{ len .Values.postgres.databases }}  # Número de elementos
```

## 🎓 Ejercicio Práctico

Modifica el template de PostgreSQL para que:

1. Use diferentes réplicas según ambiente (dev=1, staging=2, prod=3)
2. Solo habilite backups en producción
3. Use recursos diferentes por ambiente

**Solución:**
```yaml
# helmfile.d/values/postgres/values.yaml.gotmpl
{{ $env := .Environment.Name }}
{{ $isProd := eq $env "production" }}
{{ $isStaging := eq $env "staging" }}

# 1. Réplicas según ambiente
{{ if $isProd }}
replicaCount: 3
{{ else if $isStaging }}
replicaCount: 2
{{ else }}
replicaCount: 1
{{ end }}

# 2. Backups solo en prod
{{ if $isProd }}
backup:
  enabled: true
  schedule: "0 2 * * *"
  retention: 30
{{ end }}

# 3. Recursos por ambiente (mergeados automáticamente)
resources:
  {{ .Values.postgres.resources | toYaml | nindent 2 }}
```

## ✅ Checklist

- [ ] Entiendes el flujo de carga (common → env → secrets)
- [ ] Usaste `{{ .Values.* }}` para acceder a valores mergeados
- [ ] Implementaste if/else para configuración condicional
- [ ] Aplicaste with para reducir repetición
- [ ] Usaste pipelines (quote, default, toYaml, nindent)
- [ ] Probaste `helmfile ... template` en dev y production
- [ ] Usaste `write-values` para ver el merge de valores
- [ ] Viste diferencias con diff entre ambientes
- [ ] Creaste templates para app-service

## ➡️ Siguiente Paso

👉 **[04 - Multi-Ambiente](04-multi-env.md)**

Aprenderás a:
- Organizar values por ambiente
- Gestionar secrets (sin SOPS)
- Patterns de herencia de configuración
- Feature flags por ambiente
- Deploy selectivo por ambiente

---

**💡 Tip**: Usa `helmfile ... write-values` para debuggear el merge de valores. 
Te ayuda a entender qué valor final tiene cada configuración.