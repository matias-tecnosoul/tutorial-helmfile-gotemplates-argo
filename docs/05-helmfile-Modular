# 05 - Helmfile Modular (45 min)

## 🎯 Objetivo

Refactorizar el helmfile monolítico en módulos organizados por categoría, facilitando el mantenimiento y deploy selectivo.

## 📝 ¿Por Qué Modular?

### Problema: Helmfile Monolítico

```yaml
# helmfile.yaml - 500+ líneas
releases:
  - name: postgres      # Líneas 1-30
  - name: redis         # Líneas 31-60
  - name: auth-service  # Líneas 61-100
  - name: user-service  # Líneas 101-140
  - name: api-gateway   # Líneas 141-180
  - name: ingress-nginx # Líneas 181-220
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
├── 01-infrastructure.yaml  # PostgreSQL, Redis
├── 02-services.yaml        # Microservices
└── 03-ingress.yaml         # Networking
```

## 🏗️ Estructura Modular Completa

```bash
helmfile-microservices/
├── helmfile.yaml                    # Orquestador
├── helmfile.d/
│   ├── 01-infrastructure.yaml       # Bases de datos
│   ├── 02-services.yaml             # Apps del negocio
│   ├── 03-ingress.yaml              # Networking
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
│       ├── redis/
│       │   └── values.yaml.gotmpl
│       ├── auth-service/
│       │   └── values.yaml.gotmpl
│       ├── user-service/
│       │   └── values.yaml.gotmpl
│       └── api-gateway/
│           └── values.yaml.gotmpl
└── charts/                          # Charts custom
    ├── auth-service/
    ├── user-service/
    └── api-gateway/
```

## 📄 Helmfile Principal (Orquestador)

### helmfile.yaml

```yaml
# helmfile.yaml
---
# Definir ambientes UNA VEZ
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
    kubeContext: kind-helmfile-tutorial
    values:
      - helmfile.d/values/common.yaml
      - helmfile.d/environments/production/values.yaml
      - helmfile.d/environments/production/secrets.yaml

---
# Incluir helmfiles modulares EN ORDEN
helmfiles:
  - path: helmfile.d/01-infrastructure.yaml
  - path: helmfile.d/02-services.yaml
  - path: helmfile.d/03-ingress.yaml
```

## 🗄️ Módulo 1: Infraestructura

### helmfile.d/01-infrastructure.yaml

```yaml
---
# Heredar configuración de ambientes
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

## 🎯 Módulo 2: Services

### Crear charts custom

```bash
# Crear estructura de charts
mkdir -p charts/{auth-service,user-service,api-gateway}

# Auth Service
cat > charts/auth-service/Chart.yaml << 'EOF'
apiVersion: v2
name: auth-service
description: Authentication Service
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF

# User Service
cat > charts/user-service/Chart.yaml << 'EOF'
apiVersion: v2
name: user-service
description: User Management Service
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF

# API Gateway
cat > charts/api-gateway/Chart.yaml << 'EOF'
apiVersion: v2
name: api-gateway
description: API Gateway
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF
```

### Charts templates (simplificados)

```yaml
# charts/auth-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

```yaml
# charts/auth-service/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: {{ .Values.service.port }}
  selector:
    app: {{ .Release.Name }}
```

```yaml
# charts/auth-service/values.yaml (defaults)
replicaCount: 1

image:
  repository: nginx
  tag: alpine

service:
  port: 3000

env: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**Nota:** Replica la misma estructura para `user-service` y `api-gateway`.

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
  - name: auth-service
    namespace: {{ .Environment.Name }}
    chart: ../charts/auth-service
    values:
      - values/auth-service/values.yaml.gotmpl
    needs:
      - {{ .Environment.Name }}/postgres
      - {{ .Environment.Name }}/redis
    labels:
      tier: services
      component: auth
    condition: services.authService.enabled
  
  - name: user-service
    namespace: {{ .Environment.Name }}
    chart: ../charts/user-service
    values:
      - values/user-service/values.yaml.gotmpl
    needs:
      - {{ .Environment.Name }}/postgres
      - {{ .Environment.Name }}/redis
    labels:
      tier: services
      component: users
    condition: services.userService.enabled
  
  - name: api-gateway
    namespace: {{ .Environment.Name }}
    chart: ../charts/api-gateway
    values:
      - values/api-gateway/values.yaml.gotmpl
    needs:
      - {{ .Environment.Name }}/auth-service
      - {{ .Environment.Name }}/user-service
    labels:
      tier: services
      component: gateway
    condition: services.apiGateway.enabled
```

### helmfile.d/values/common.yaml (actualizado)

```yaml
---
# Services configuration
services:
  authService:
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
  
  userService:
    enabled: true
    image:
      repository: nginx
      tag: alpine
    replicaCount: 1
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
  
  apiGateway:
    enabled: true
    image:
      repository: nginx
      tag: alpine
    replicaCount: 1
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

### helmfile.d/values/auth-service/values.yaml.gotmpl

```yaml
---
{{ $env := .Environment.Name }}

replicaCount: {{ .Values.services.authService.replicaCount }}

image:
  repository: {{ .Values.services.authService.image.repository }}
  tag: {{ .Values.services.authService.image.tag }}

service:
  port: 3000

env:
  - name: NODE_ENV
    value: {{ $env }}
  - name: DATABASE_HOST
    value: postgres.{{ $env }}.svc.cluster.local
  - name: REDIS_HOST
    value: redis.{{ $env }}.svc.cluster.local
  - name: SERVICE_NAME
    value: auth-service

resources:
  {{ .Values.services.authService.resources | toYaml | nindent 2 }}
```

**Replica para `user-service` y `api-gateway` con valores correspondientes.**

## 🌐 Módulo 3: Ingress

### helmfile.d/03-ingress.yaml

```yaml
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
    labels:
      tier: networking
      component: ingress
    condition: ingressNginx.enabled
    needs:
      - {{ .Environment.Name }}/auth-service
      - {{ .Environment.Name }}/user-service
      - {{ .Environment.Name }}/api-gateway
```

### helmfile.d/values/common.yaml (agregar)

```yaml
# Ingress
ingressNginx:
  enabled: true
```

### helmfile.d/values/nginx-ingress/values.yaml.gotmpl

```yaml
---
controller:
  replicaCount: 1
  
  service:
    type: NodePort
    nodePorts:
      http: 30080
      https: 30443
  
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

## 🎮 Comandos Modulares

### Listar por módulo

```bash
# Toda la infraestructura
helmfile -e dev -f helmfile.d/01-infrastructure.yaml list

# Todos los servicios
helmfile -e dev -f helmfile.d/02-services.yaml list

# Solo ingress
helmfile -e dev -f helmfile.d/03-ingress.yaml list
```

### Deploy selectivo

```bash
# Solo infraestructura
helmfile -e dev -f helmfile.d/01-infrastructure.yaml apply

# Solo servicios (requiere infraestructura)
helmfile -e dev -f helmfile.d/02-services.yaml apply

# Solo ingress
helmfile -e dev -f helmfile.d/03-ingress.yaml apply
```

### Deploy por labels

```bash
# Solo databases
helmfile -e dev -l component=database apply

# Solo cache
helmfile -e dev -l component=cache apply

# Toda la infraestructura
helmfile -e dev -l tier=infrastructure apply

# Todos los servicios
helmfile -e dev -l tier=services apply
```

### Deploy completo

```bash
# Todo en orden (respeta helmfiles: order)
helmfile -e dev apply
```

## 📊 Flujo de Deploy

```
helmfile apply
    ↓
helmfile.d/01-infrastructure.yaml
    ↓
  postgres (deployed)
    ↓
  redis (deployed)
    ↓
helmfile.d/02-services.yaml
    ↓
  auth-service (needs: postgres, redis)
    ↓
  user-service (needs: postgres, redis)
    ↓
  api-gateway (needs: auth-service, user-service)
    ↓
helmfile.d/03-ingress.yaml
    ↓
  ingress-nginx (needs: all services)
```

## 🧪 Verificar Deploy Modular

```bash
# 1. Infraestructura
helmfile -e dev -f helmfile.d/01-infrastructure.yaml apply

kubectl get pods -n dev
# Debe mostrar: postgres-0, redis-xxx

# 2. Services
helmfile -e dev -f helmfile.d/02-services.yaml apply

kubectl get pods -n dev
# Debe mostrar: auth-service-xxx, user-service-xxx, api-gateway-xxx

# 3. Ingress
helmfile -e dev -f helmfile.d/03-ingress.yaml apply

kubectl get pods -n ingress-nginx
# Debe mostrar: ingress-nginx-controller-xxx
```

## 🎯 Patrones de Organización

### Por tipo de recurso (Mikroways)

```
helmfile.d/
├── 01-infrastructure.yaml    # DB, cache
├── 02-services.yaml          # Apps
├── 03-ingress.yaml           # Networking
```

### Por dominio de negocio

```
helmfile.d/
├── 01-shared.yaml            # Infra compartida
├── 02-auth-domain.yaml       # Auth + related
├── 03-user-domain.yaml       # Users + related
├── 04-gateway.yaml           # API Gateway
```

### Por criticidad

```
helmfile.d/
├── 01-critical.yaml          # Core services
├── 02-standard.yaml          # Normal priority
├── 03-optional.yaml          # Nice to have
```

## 📝 Convenciones de Numeración

```
01-  Base layer (databases, cache)
02-  Application layer (business logic)
03-  Presentation layer (ingress, API gw)
04-  Observability (monitoring, logging)
05-  Security (policies, scanners)
```

## 🔄 Ventajas vs Desventajas

### ✅ Ventajas

- Separación clara de responsabilidades
- Deploy selectivo por capa
- Menos conflictos de Git
- Fácil onboarding (ver solo lo relevante)
- Escalable (20+ releases)

### ⚠️ Desventajas

- Más archivos que gestionar
- Paths relativos (`../values/`)
- Overhead para proyectos pequeños (<5 releases)
- Duplicación de configuración de ambientes

## 🐛 Troubleshooting

### Paths relativos incorrectos

```yaml
# ❌ ERROR (desde helmfile.d/01-infrastructure.yaml)
values:
  - values/postgres/values.yaml.gotmpl

# ✅ CORRECTO
values:
  - values/postgres/values.yaml.gotmpl
# (paths son relativos al helmfile que los define)
```

### Ambientes no heredados

```bash
# Cada helmfile modular debe declarar environments
# Helmfile no hereda automáticamente del principal
```

### Dependencies entre módulos

```yaml
# ❌ No funciona cross-helmfile automático
needs:
  - postgres  # Solo funciona en mismo helmfile

# ✅ Usar namespace prefix
needs:
  - {{ .Environment.Name }}/postgres
```

## ✅ Checklist

- [ ] Creaste helmfile.d/ con 01, 02, 03
- [ ] helmfile.yaml orquesta los módulos
- [ ] Cada módulo declara sus environments
- [ ] Paths relativos funcionan correctamente
- [ ] Deploy selectivo por módulo funciona
- [ ] Deploy completo respeta orden
- [ ] Dependencies cross-module funcionan

## ➡️ Siguiente Paso

👉 **[06 - Dependencias](06-dependencies.md)**

Aprenderás:
- Dependencias con `needs:`
- Orden de ejecución
- Wait conditions
- Dependencias condicionales

---

**💡 Tip**: Para proyectos <5 releases, un solo helmfile.yaml es suficiente. Usa helmfile.d/ cuando el proyecto crece.