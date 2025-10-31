# 06 - Dependencias (30 min)

## 🎯 Objetivo

Gestionar el orden de despliegue usando dependencias explícitas, asegurando que los servicios se desplieguen en el orden correcto.

## 📝 El Problema

```bash
# Sin dependencias:
API Gateway arranca → Error: auth-service no existe
Auth Service arranca → Error: postgres no responde
PostgreSQL arranca → OK (pero tarde)

# Resultado: CrashLoopBackOff 💥
```

## 💡 La Solución: needs

```yaml
releases:
  - name: postgres
    # No necesita nada
  
  - name: auth-service
    needs:
      - {{ .Environment.Name }}/postgres  # Espera a postgres
  
  - name: api-gateway
    needs:
      - {{ .Environment.Name }}/auth-service  # Espera a auth
```

## 🏗️ Dependencias Básicas

### helmfile.d/01-infrastructure.yaml

```yaml
---
releases:
  - name: postgres
    namespace: {{ .Environment.Name }}
    chart: groundhog2k/postgres
    values:
      - values/postgres/values.yaml.gotmpl
    labels:
      tier: infrastructure
      component: database
  
  - name: redis
    namespace: {{ .Environment.Name }}
    chart: groundhog2k/redis
    values:
      - values/redis/values.yaml.gotmpl
    labels:
      tier: infrastructure
      component: cache
    # Redis no depende de postgres
```

### helmfile.d/02-services.yaml

```yaml
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
```

## 📊 Grafo de Dependencias

```
         postgres
            ↓
    ┌───────┴───────┐
    ↓               ↓
auth-service   user-service
    ↓               ↓
    └───────┬───────┘
            ↓
       api-gateway
            ↓
      ingress-nginx
```

## 🔗 Dependencias Cross-Module

### helmfile.d/03-ingress.yaml

```yaml
---
releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    values:
      - values/nginx-ingress/values.yaml.gotmpl
    needs:
      # Esperar a que todos los servicios estén listos
      - {{ .Environment.Name }}/auth-service
      - {{ .Environment.Name }}/user-service
      - {{ .Environment.Name }}/api-gateway
    labels:
      tier: networking
```

## 🧪 Verificar Orden de Ejecución

### Deploy y observar orden

```bash
# Deploy con verbose para ver orden
helmfile -e dev apply --debug 2>&1 | grep "Upgrading release"
```

**Salida esperada:**
```
Upgrading release=postgres
Upgrading release=redis
Upgrading release=auth-service
Upgrading release=user-service
Upgrading release=api-gateway
Upgrading release=ingress-nginx
```

### Watch en tiempo real

```bash
# Terminal 1: Deploy
helmfile -e dev apply

# Terminal 2: Watch pods
watch kubectl get pods -n dev
```

**Observarás:**
1. postgres-0 y redis-xxx primero
2. auth-service-xxx y user-service-xxx después
3. api-gateway-xxx al final

## ⏱️ Wait y Timeout

### Configuración de wait

```yaml
# helmfile.d/01-infrastructure.yaml
releases:
  - name: postgres
    namespace: {{ .Environment.Name }}
    chart: groundhog2k/postgres
    values:
      - values/postgres/values.yaml.gotmpl
    wait: true              # Esperar a que esté ready
    timeout: 300            # 5 minutos máximo
    waitForJobs: true       # Esperar jobs (migraciones, etc.)
```

### Por defecto en helmfile.yaml

```yaml
# helmfile.yaml
---
# Configuración global de wait
helmDefaults:
  wait: true
  timeout: 300
  waitForJobs: true
  atomic: true  # Rollback automático si falla

environments:
  dev:
    values: [...]
```

## 🎯 Dependencias Condicionales

### Depender solo si está habilitado

```yaml
# helmfile.d/02-services.yaml
releases:
  - name: api-gateway
    namespace: {{ .Environment.Name }}
    chart: ../charts/api-gateway
    values:
      - values/api-gateway/values.yaml.gotmpl
    needs:
      - {{ .Environment.Name }}/auth-service
      - {{ .Environment.Name }}/user-service
      # Solo depender de Redis si está habilitado
      {{- if .Values.redis.enabled }}
      - {{ .Environment.Name }}/redis
      {{- end }}
```

### Feature flags con dependencias

```yaml
# environments/dev/values.yaml
features:
  monitoring: false
  backup: false

# helmfile.d/02-services.yaml
releases:
  - name: auth-service
    needs:
      - {{ .Environment.Name }}/postgres
      {{- if .Values.features.monitoring }}
      - monitoring/prometheus
      {{- end }}
```

## 🔄 Dependencias Circulares (Evitar)

### ❌ Problema

```yaml
# Service A depende de B
- name: service-a
  needs:
    - {{ .Environment.Name }}/service-b

# Service B depende de A
- name: service-b
  needs:
    - {{ .Environment.Name }}/service-a

# ERROR: Circular dependency
```

### ✅ Solución

```yaml
# Refactorizar para eliminar ciclo
- name: shared-config
  # ConfigMap compartido

- name: service-a
  needs:
    - {{ .Environment.Name }}/shared-config

- name: service-b
  needs:
    - {{ .Environment.Name }}/shared-config
```

## 🎨 Patrones Avanzados

### Dependencias en paralelo

```yaml
releases:
  - name: postgres
    # Base
  
  # Estos se despliegan en paralelo (ambos dependen solo de postgres)
  - name: auth-service
    needs:
      - {{ .Environment.Name }}/postgres
  
  - name: user-service
    needs:
      - {{ .Environment.Name }}/postgres
  
  # Este espera a ambos
  - name: api-gateway
    needs:
      - {{ .Environment.Name }}/auth-service
      - {{ .Environment.Name }}/user-service
```

### Dependencias opcionales

```yaml
- name: api-gateway
  needs:
    # Siempre necesita estos
    - {{ .Environment.Name }}/auth-service
    - {{ .Environment.Name }}/user-service
    
    # Opcional según ambiente
    {{ if eq .Environment.Name "production" }}
    - {{ .Environment.Name }}/redis-sentinel
    {{ else }}
    - {{ .Environment.Name }}/redis
    {{ end }}
```

## 📦 Dependencias Externas

### Depender de releases fuera del helmfile

```yaml
releases:
  - name: my-app
    needs:
      # Release instalado manualmente o por otro helmfile
      - kube-system/cert-manager
      - monitoring/prometheus-operator
```

## 🐛 Troubleshooting

### Dependencia no encontrada

```bash
# Error:
# release "postgres" not found in namespace "dev"

# Solución: Verificar namespace
needs:
  - {{ .Environment.Name }}/postgres  # ✅ Con namespace
```

### Timeout esperando dependencia

```bash
# Error:
# timed out waiting for the condition

# Solución 1: Aumentar timeout
timeout: 600  # 10 minutos

# Solución 2: Verificar que el pod arranca
kubectl describe pod -n dev postgres-0

# Solución 3: Ver logs
kubectl logs -n dev postgres-0
```

### Dependencias en orden incorrecto

```bash
# Ver orden planificado (sin aplicar)
helmfile -e dev list

# Verificar needs de cada release
helmfile -e dev list | grep -A 5 "name:"
```

## 🎯 Testing de Dependencias

### Eliminar todo y re-deployar

```bash
# Eliminar todo
helmfile -e dev destroy

# Deploy desde cero
helmfile -e dev apply

# Verificar que el orden fue correcto
kubectl get events -n dev --sort-by='.lastTimestamp' | grep Created
```

**Debe mostrar:**
1. postgres primero
2. redis después o en paralelo
3. services después
4. gateway al final

### Deploy solo un release (ignora needs)

```bash
# Forzar deploy sin esperar dependencias
helmfile -e dev -l name=api-gateway sync --skip-needs

# ⚠️ Puede fallar si las dependencias no existen
```

## 📊 Visualizar Dependencias

### Listar releases con sus needs

```bash
helmfile -e dev list --output json | jq '.[] | {name: .name, needs: .needs}'
```

**Salida:**
```json
{
  "name": "postgres",
  "needs": null
}
{
  "name": "auth-service",
  "needs": ["dev/postgres", "dev/redis"]
}
{
  "name": "api-gateway",
  "needs": ["dev/auth-service", "dev/user-service"]
}
```

## 🎓 Best Practices

### 1. Namespace explícito en needs

```yaml
# ❌ Ambiguo
needs:
  - postgres

# ✅ Explícito
needs:
  - {{ .Environment.Name }}/postgres
```

### 2. Wait en bases de datos

```yaml
- name: postgres
  wait: true
  timeout: 300
  # Asegura que está listo antes de continuar
```

### 3. Dependencias mínimas

```yaml
# ❌ Dependencias innecesarias
- name: api-gateway
  needs:
    - postgres       # No usa postgres directamente
    - redis         # No usa redis directamente
    - auth-service  # ✅ Sí lo usa
    - user-service  # ✅ Sí lo usa

# ✅ Solo lo necesario
- name: api-gateway
  needs:
    - auth-service
    - user-service
```

### 4. Documentar dependencias complejas

```yaml
- name: api-gateway
  # Depende de auth y user porque:
  # - auth: validación de tokens
  # - user: datos de usuario
  needs:
    - {{ .Environment.Name }}/auth-service
    - {{ .Environment.Name }}/user-service
```

## ✅ Checklist

- [ ] Agregaste `needs:` a todos los releases apropiados
- [ ] Usaste formato `{{ .Environment.Name }}/release-name`
- [ ] Configuraste `wait: true` en bases de datos
- [ ] Evitaste dependencias circulares
- [ ] Testeaste deploy desde cero
- [ ] Verificaste orden con `kubectl get events`
- [ ] Deploy completo funciona sin errores

## ➡️ Siguiente Paso

👉 **[07 - Ingress](07-ingress.md)**

Aprenderás:
- Deploy de Nginx Ingress Controller
- Crear Ingress resources para cada service
- Templating de hosts por ambiente
- Testing de endpoints

---

**💡 Tip**: Usa `wait: true` y `timeout` generosos en producción. Es mejor esperar que tener un deploy parcialmente fallido.