# 06 - Dependencias (30 min)

## 🎯 Objetivo

Gestionar el orden de despliegue usando dependencias explícitas, asegurando que los servicios se desplieguen en el orden correcto.

## 📝 El Problema
```bash
# Sin dependencias:
app-service arranca → Error: postgres no responde
PostgreSQL arranca → OK (pero tarde)

# Resultado: CrashLoopBackOff 💥
```

**¿Por qué pasa esto?**

Kubernetes y Helm no garantizan orden de despliegue. Si desplegamos todo a la vez, app-service puede arrancar antes que PostgreSQL esté listo.

## 💡 La Solución: needs
```yaml
releases:
  - name: postgres
    # No necesita nada
  
  - name: app-service
    needs:
      - dev/postgres  # Espera a postgres
```

Helmfile esperará a que `postgres` esté desplegado antes de instalar `app-service`.

## 🏗️ Dependencias Básicas

### helmfile.d/01-infrastructure.yaml
```yaml
---
releases:
  - name: postgres
    namespace: dev
    chart: groundhog2k/postgres
    values:
      - values/postgres/values.yaml.gotmpl
    wait: true
    timeout: 300
    labels:
      tier: infrastructure
      component: database
```

**Sin dependencias** - PostgreSQL es la base, no depende de nada.

### helmfile.d/02-services.yaml
```yaml
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
      - dev/postgres  # ← Dependencia explícita
    labels:
      tier: services
      component: app
```

**Con dependencias** - app-service necesita que PostgreSQL esté listo.

## 📊 Grafo de Dependencias
```
         postgres
            ↓
       app-service
            ↓
      ingress-nginx (OPCIONAL)
```

**Flujo de deploy:**
1. PostgreSQL se instala primero
2. Helmfile espera a que esté ready (wait: true)
3. app-service se instala después
4. (Opcional) Ingress se instala al final

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
    wait: true
    timeout: 300
    needs:
      # Esperar a que app-service esté listo
      - dev/app-service
    labels:
      tier: networking
```

**Dependencias entre módulos** - Ingress espera a que app-service (de otro módulo) esté listo.

## 🧪 Verificar Orden de Ejecución

### Deploy y observar orden
```bash
# Terminal 1: Deploy infraestructura
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# Terminal 2: Watch pods (ejecutar antes del apply)
watch kubectl get pods -n dev

# Verás:
# 1. postgres-0 arranca primero
# 2. Pasa a Running/Ready
# 3. Solo entonces continúa
```

### Deploy services con dependencia
```bash
# Terminal 1: Deploy services
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Terminal 2: Watch pods
watch kubectl get pods -n dev

# Verás:
# 1. Helmfile verifica que postgres existe
# 2. app-service arranca
# 3. app-service se conecta a postgres exitosamente
```

### Verificar logs
```bash
# Ver logs de app-service
kubectl logs -n dev -l app=app-service -f

# Deberías ver:
# ✅ Database table "tasks" ready
# ✅ Sample tasks inserted
# 🚀 App service listening on port 3000
```

## ⏱️ Wait y Timeout

### Configuración de wait
```yaml
# helmfile.d/01-infrastructure.yaml
releases:
  - name: postgres
    namespace: dev
    chart: groundhog2k/postgres
    values:
      - values/postgres/values.yaml.gotmpl
    wait: true              # Esperar a que esté ready
    timeout: 300            # 5 minutos máximo
```

**¿Qué hace `wait: true`?**

Helmfile espera a que:
- El pod esté en estado `Running`
- Los readiness probes pasen
- El rollout esté completo

### Timeouts apropiados
```yaml
# Base de datos (puede tardar en arrancar)
- name: postgres
  wait: true
  timeout: 300  # 5 minutos

# Aplicación (arranque rápido)
- name: app-service
  wait: true
  timeout: 180  # 3 minutos

# Ingress controller (puede tardar)
- name: ingress-nginx
  wait: true
  timeout: 300  # 5 minutos
```

## 🎯 Testing de Dependencias

### Test 1: Deploy desde cero
```bash
# Eliminar todo
helmfile -f helmfile.d/01-infrastructure.yaml -e dev destroy
helmfile -f helmfile.d/02-services.yaml -e dev destroy

# Verificar que no hay nada
kubectl get all -n dev

# Deploy en orden correcto
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Verificar orden en eventos
kubectl get events -n dev --sort-by='.lastTimestamp' | grep Created
```

**Salida esperada:**
```
# PostgreSQL primero
2m    Normal   Created    pod/postgres-0    Created container postgres

# app-service después
1m    Normal   Created    pod/app-service-xxx    Created container app-service
```

### Test 2: Intentar deploy sin dependencias
```bash
# Eliminar todo
helmfile -f helmfile.d/01-infrastructure.yaml -e dev destroy
helmfile -f helmfile.d/02-services.yaml -e dev destroy

# Intentar deploy de services sin infra
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Resultado:
# Error: release "postgres" in namespace "dev" not found
# Helmfile detiene el deploy (gracias a needs:)
```

### Test 3: Conectividad de app-service
```bash
# Deploy completo
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Port-forward
kubectl port-forward -n dev svc/app-service 3000:80

# Probar endpoint de health
curl http://localhost:3000/health

# Debe mostrar:
# {
#   "status": "healthy",
#   "db": "connected",  ← Importante: conectado a postgres
#   "version": "1.0.0"
# }

# Probar API de tasks (usa postgres)
curl http://localhost:3000/api/tasks

# Debe mostrar las 2 tareas de ejemplo
```

## 📝 Formato de needs

### Sintaxis correcta
```yaml
# ✅ CORRECTO - Con namespace
needs:
  - dev/postgres

# ✅ CORRECTO - Múltiples dependencias
needs:
  - dev/postgres
  - dev/redis

# ❌ ERROR - Sin namespace
needs:
  - postgres  # No funciona cross-module

# ❌ ERROR - Namespace incorrecto
needs:
  - default/postgres  # Namespace equivocado
```

### Dependencias del mismo módulo
```yaml
# Si tienes múltiples releases en el mismo módulo
releases:
  - name: postgres
    namespace: dev
  
  - name: postgres-backup
    namespace: dev
    needs:
      - dev/postgres  # Mismo módulo, necesita namespace
```

## 🐛 Troubleshooting

### Dependencia no encontrada
```bash
# Error:
# release "postgres" in namespace "dev" not found

# Causa: needs apunta a namespace/release incorrecto
# Solución: Verificar formato
needs:
  - dev/postgres  # namespace/release
```

### Timeout esperando dependencia
```bash
# Error:
# timed out waiting for the condition

# Causa 1: Pod no arranca
kubectl describe pod -n dev postgres-0

# Causa 2: Readiness probe falla
kubectl logs -n dev postgres-0

# Causa 3: Timeout muy corto
# Solución: Aumentar timeout
timeout: 600  # 10 minutos
```

### App-service falla al conectar
```bash
# Ver logs
kubectl logs -n dev -l app=app-service

# Error común:
# Error: connect ECONNREFUSED postgres.dev.svc.cluster.local:5432

# Verificar:
# 1. PostgreSQL está running
kubectl get pods -n dev -l app.kubernetes.io/name=postgres

# 2. Service existe
kubectl get svc -n dev postgres

# 3. DNS resuelve
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup postgres.dev.svc.cluster.local

# 4. Puerto correcto en app-service values
# DB_HOST: postgres.dev.svc.cluster.local
# DB_PORT: 5432
```

### Deploy se queda esperando
```bash
# Helmfile se queda en "Waiting for release..."

# Ver qué está pasando
kubectl get pods -n dev -w

# Ver eventos
kubectl get events -n dev --sort-by='.lastTimestamp'

# Verificar logs del pod que no arranca
kubectl logs -n dev <pod-name>
```

## 🎓 Best Practices

### 1. Namespace explícito en needs
```yaml
# ❌ Ambiguo
needs:
  - postgres

# ✅ Explícito
needs:
  - dev/postgres
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
- name: app-service
  needs:
    - dev/postgres        # ✅ Sí lo usa
    - dev/some-configmap  # ❌ No lo necesita

# ✅ Solo lo necesario
- name: app-service
  needs:
    - dev/postgres
```

### 4. Documentar dependencias
```yaml
- name: app-service
  # Depende de postgres porque:
  # - Necesita DB para tasks
  # - Connection string apunta a postgres service
  needs:
    - dev/postgres
```

### 5. Timeout generoso en producción
```yaml
# Desarrollo
timeout: 180  # 3 min OK

# Producción
timeout: 600  # 10 min (imágenes grandes, init containers, etc.)
```

## 📚 Comparación: Con vs Sin Dependencias

### Sin needs
```yaml
# helmfile.d/02-services.yaml
releases:
  - name: app-service
    namespace: dev
    chart: ../charts/app-service
    # Sin needs
```

**Resultado:**
```bash
helmfile -f helmfile.d/02-services.yaml -e dev apply

# app-service intenta arrancar inmediatamente
# CrashLoopBackOff si postgres no está listo
# Requiere intervención manual
```

### Con needs
```yaml
# helmfile.d/02-services.yaml
releases:
  - name: app-service
    namespace: dev
    chart: ../charts/app-service
    needs:
      - dev/postgres  # ← Magia aquí
```

**Resultado:**
```bash
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Helmfile verifica que postgres existe
# Helmfile espera a que postgres esté ready
# app-service arranca solo cuando postgres está listo
# ✅ Sin CrashLoopBackOff
```

## 🎓 Ejercicio Práctico

**Objetivo:** Ver qué pasa sin dependencias.
```bash
# 1. Eliminar needs de app-service
nano helmfile.d/02-services.yaml
# Comentar línea: needs: - dev/postgres

# 2. Eliminar todo
helmfile -f helmfile.d/01-infrastructure.yaml -e dev destroy
helmfile -f helmfile.d/02-services.yaml -e dev destroy

# 3. Deploy services ANTES de infra (orden incorrecto)
helmfile -f helmfile.d/02-services.yaml -e dev apply
# Arranca sin esperar postgres

# 4. Ver el error
kubectl logs -n dev -l app=app-service
# Error: connect ECONNREFUSED

# 5. Deploy infra (ahora sí)
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply

# 6. Esperar y verificar
# app-service se auto-recupera cuando postgres esté listo
# (Kubernetes reinicia el pod automáticamente)

# 7. Restaurar needs
nano helmfile.d/02-services.yaml
# Descomentar: needs: - dev/postgres
```

**Lección:** `needs:` evita estos problemas desde el inicio.

## ✅ Checklist

- [ ] Agregaste `needs:` a app-service apuntando a postgres
- [ ] Usaste formato `dev/postgres` (con namespace)
- [ ] Configuraste `wait: true` en PostgreSQL
- [ ] Configuraste timeout apropiado (300s)
- [ ] Testeaste deploy desde cero
- [ ] Verificaste logs de app-service (conexión OK)
- [ ] Verificaste orden con `kubectl get events`
- [ ] Entiendes diferencia entre con/sin needs

## ➡️ Siguiente Paso

👉 **[07 - Ingress (OPCIONAL)](07-ingress.md)**

**⚠️ Este capítulo es OPCIONAL**

Para el flujo principal del tutorial, puedes usar `kubectl port-forward`:
```bash
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/api/tasks
```

Si quieres aprender sobre Ingress Controller y exponer aplicaciones con hosts dinámicos, continúa con el capítulo 07.

---

**💡 Tip**: Usa `wait: true` y timeout generoso en producción. Es mejor esperar que tener un deploy parcialmente fallido.