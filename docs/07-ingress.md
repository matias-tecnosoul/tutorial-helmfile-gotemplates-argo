# 07 - Ingress (OPCIONAL/BONUS) (30 min)

## ⚠️ ESTE CAPÍTULO ES OPCIONAL

Este capítulo enseña cómo exponer aplicaciones usando Nginx Ingress Controller.

**Para el flujo principal del tutorial**, puedes usar `kubectl port-forward`:
```bash
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/api/tasks
```

**Si quieres aprender sobre Ingress**, continúa con este capítulo.

---

## 🎯 Objetivo

Exponer app-service mediante Nginx Ingress Controller, con configuración dinámica por ambiente.

## 📝 ¿Qué es Ingress?

**Problema sin Ingress:**
```bash
# Cada service necesita LoadBalancer o NodePort
service/app-service    LoadBalancer  35.xxx.xxx.xxx  # $$$
```

**Solución con Ingress:**
```bash
# Un solo LoadBalancer, múltiples rutas
ingress-nginx-controller  LoadBalancer  35.xxx.xxx.xxx

app.example.com     → app-service:80
```

**Ventajas:**
- Un solo punto de entrada
- Routing basado en host/path
- TLS/HTTPS centralizado
- Menos costos (1 LB en vez de N)

## 🏗️ Nginx Ingress Controller

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

### helmfile.d/values/nginx-ingress/values.yaml.gotmpl
```yaml
---
{{ $env := .Environment.Name }}
{{ $isProd := eq $env "production" }}

controller:
  # Réplicas según ambiente
  {{ if $isProd }}
  replicaCount: 3
  {{ else }}
  replicaCount: 1
  {{ end }}
  
  # Service type (NodePort para Kind)
  service:
    type: NodePort
    nodePorts:
      http: 30080
      https: 30443
  
  # Recursos
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      {{ if $isProd }}
      cpu: 1000m
      memory: 512Mi
      {{ else }}
      cpu: 500m
      memory: 256Mi
      {{ end }}

# Default backend
defaultBackend:
  enabled: true
```

## 🌐 Ingress Resource en Chart

El chart de app-service ya tiene el template de Ingress.

### charts/app-service/templates/ingress.yaml
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
  {{- range .Values.ingress.tls }}
  - hosts:
    {{- range .hosts }}
    - {{ . | quote }}
    {{- end }}
    secretName: {{ .secretName }}
  {{- end }}
  {{- end }}
  rules:
  {{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
    http:
      paths:
      {{- range .paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
        backend:
          service:
            name: {{ $.Release.Name }}
            port:
              number: {{ $.Values.service.port }}
      {{- end }}
  {{- end }}
{{- end }}
```

## 🎨 Habilitar Ingress en Values

### helmfile.d/values/common.yaml (actualizar)
```yaml
---
# Global
baseDomain: example.com

# Ingress (deshabilitado por defecto)
ingressNginx:
  enabled: false  # Cambiar a true para habilitar

# App Service
appService:
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
  ingress:
    enabled: false  # Cambiar a true para habilitar
```

### helmfile.d/environments/dev/values.yaml (actualizar)
```yaml
---
baseDomain: dev.example.local

# Habilitar ingress en dev (OPCIONAL)
ingressNginx:
  enabled: true

appService:
  ingress:
    enabled: true
```

### helmfile.d/values/app-service/values.yaml.gotmpl (ya tiene ingress)
```yaml
---
{{ $env := .Environment.Name }}

# ... (resto del template)

# Ingress dinámico
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

## 🚀 Deploy con Ingress

### 1. Habilitar Ingress
```bash
# Editar values
nano helmfile.d/values/common.yaml

# Cambiar:
ingressNginx:
  enabled: true  # ← De false a true
```
```bash
# Editar environment
nano helmfile.d/environments/dev/values.yaml

# Agregar/actualizar:
ingressNginx:
  enabled: true

appService:
  ingress:
    enabled: true
```

### 2. Deploy Ingress Controller
```bash
# Deploy ingress controller
helmfile -f helmfile.d/03-ingress.yaml -e dev apply

# Verificar
kubectl get pods -n ingress-nginx
```

**Salida esperada:**
```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
ingress-nginx-defaultbackend-xxxxx-xxxxx    1/1     Running   0          1m
```

### 3. Re-deploy app-service con Ingress habilitado
```bash
# Re-deploy app-service
helmfile -f helmfile.d/02-services.yaml -e dev apply

# Verificar ingress resource
kubectl get ingress -n dev
```

**Salida esperada:**
```
NAME          CLASS   HOSTS                    ADDRESS   PORTS   AGE
app-service   nginx   app.dev.example.local              80      30s
```

## 🧪 Testing de Endpoints

### Opción 1: Port-forward al Ingress Controller
```bash
# Terminal 1: Port-forward
kubectl port-forward -n ingress-nginx \
  svc/ingress-nginx-controller 8080:80

# Terminal 2: Test con Host header
curl -H "Host: app.dev.example.local" http://localhost:8080/health
curl -H "Host: app.dev.example.local" http://localhost:8080/api/tasks
```

**Salida esperada:**
```json
{
  "status": "healthy",
  "db": "connected",
  "version": "1.0.0"
}
```

### Opción 2: Hosts locales (para usar navegador)
```bash
# Agregar a /etc/hosts
sudo nano /etc/hosts

# Agregar línea:
127.0.0.1 app.dev.example.local
```

Luego, con port-forward activo:
```bash
# En navegador o curl sin -H
firefox http://app.dev.example.local:8080
curl http://app.dev.example.local:8080/health
```

### Opción 3: NodePort directo (Kind)
```bash
# Kind expone NodePort en el host
# Según kind-config.yaml: puerto 80 del host → 30080 del nodo

# Agregar a /etc/hosts
sudo nano /etc/hosts
127.0.0.1 app.dev.example.local

# Acceder directamente (sin port-forward)
curl -H "Host: app.dev.example.local" http://localhost:80/health

# O en navegador
firefox http://app.dev.example.local
```

## 🎯 Configuraciones Avanzadas

### Rate limiting
```yaml
# environments/production/values.yaml
appService:
  ingress:
    enabled: true
    annotations:
      nginx.ingress.kubernetes.io/rate-limit: "100"
      nginx.ingress.kubernetes.io/limit-rps: "10"
```

### CORS
```yaml
appService:
  ingress:
    enabled: true
    annotations:
      nginx.ingress.kubernetes.io/enable-cors: "true"
      nginx.ingress.kubernetes.io/cors-allow-origin: "*"
      nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
```

### Timeouts
```yaml
appService:
  ingress:
    enabled: true
    annotations:
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
```

### TLS/HTTPS (producción)
```yaml
# environments/production/values.yaml
appService:
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
      - secretName: app-service-tls
        hosts:
          - app.example.com
```

> 💡 **Nota:** Requiere cert-manager instalado en el cluster.

## 🐛 Troubleshooting

### Ingress no responde (404)
```bash
# Verificar ingress resource
kubectl describe ingress -n dev app-service

# Verificar backend service
kubectl get svc -n dev app-service

# Ver logs del controller
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f
```

### Service backend no encontrado
```bash
# Error común: service name incorrecto en ingress

# Verificar que el service existe
kubectl get svc -n dev

# Verificar endpoints
kubectl get endpoints -n dev app-service

# Debe tener IPs de pods
# NAME          ENDPOINTS         AGE
# app-service   10.244.0.5:3000   2m
```

### Ingress controller no arranca
```bash
# Ver estado
kubectl get pods -n ingress-nginx

# Ver logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Error común en Kind: puertos ya en uso
# Solución: Cambiar NodePort en values o liberar puertos del host
```

### Host header no funciona
```bash
# Verificar que usas el host correcto
kubectl get ingress -n dev app-service -o yaml | grep host

# Debe coincidir con tu curl:
curl -H "Host: app.dev.example.local" http://localhost:8080/
```

## 📝 Ingress vs Service Types

| Tipo | Cuándo Usar | Costo | Ejemplo |
|------|-------------|-------|---------|
| **ClusterIP** | Interno solo | Gratis | DB, cache |
| **NodePort** | Dev/testing | Gratis | Kind clusters |
| **LoadBalancer** | Producción (sin ingress) | $$$ | 1 service = 1 LB |
| **Ingress** | Producción (múltiples apps) | $ | 1 LB para N apps |

**Recomendación:**
- **Dev local:** Port-forward (más simple) o Ingress (más realista)
- **Staging/Prod:** Ingress + LoadBalancer

## 🎓 Comparación: Port-forward vs Ingress

### Port-forward (recomendado para dev)
```bash
kubectl port-forward -n dev svc/app-service 3000:80
curl http://localhost:3000/health
```

**✅ Ventajas:**
- Simple
- No requiere configuración adicional
- Funciona en cualquier cluster

**❌ Desventajas:**
- Solo una terminal a la vez
- No routing basado en host
- No TLS

### Ingress (opcional para dev, necesario para prod)
```bash
curl -H "Host: app.dev.example.local" http://localhost:8080/health
```

**✅ Ventajas:**
- Routing por host/path
- TLS/HTTPS
- Más cercano a producción
- Rate limiting, CORS, etc.

**❌ Desventajas:**
- Más complejo
- Requiere DNS/hosts
- Más componentes a mantener

## ✅ Checklist

- [ ] Entiendes qué es Ingress y cuándo usarlo
- [ ] (Opcional) Desplegaste Nginx Ingress Controller
- [ ] (Opcional) Habilitaste ingress en app-service values
- [ ] (Opcional) Creaste Ingress resource
- [ ] (Opcional) Probaste con curl usando Host header
- [ ] Entiendes diferencia entre port-forward e Ingress
- [ ] Sabes que Ingress es OPCIONAL para este tutorial

## 🎯 Para Continuar Aprendiendo

Este capítulo es opcional. Has completado el core del tutorial de Helmfile + Go Templates.

**Stack desplegado:**
- ✅ PostgreSQL (infraestructura)
- ✅ app-service (aplicación)
- ✅ (Opcional) Nginx Ingress Controller

**Lo que aprendiste:**
1. ✅ Helmfile básico y modular
2. ✅ Go Templates (variables, condicionales, pipelines)
3. ✅ Multi-ambiente (dev/staging/production)
4. ✅ Organización modular (helmfile.d/)
5. ✅ Dependencias (needs, wait, timeout)
6. ✅ (Opcional) Ingress Controller

## 🚀 Próximos Tutoriales

Has completado "Helmfile + Go Templates". Continúa tu aprendizaje con:

### 1. GitOps con Helmfile + ArgoCD
- Despliegue automático desde Git
- Drift detection y auto-heal
- Rollbacks y historial
- Multi-cluster management

**[Tutorial separado - Próximamente]**

### 2. Secrets Management
- SOPS + Age encryption
- Sealed Secrets
- External Secrets Operator
- Best practices de seguridad

### 3. Observability Stack
- Prometheus + Grafana
- Loki para logs
- Integración con Helmfile

---

## 🎉 ¡Felicitaciones!

Has completado el tutorial de Helmfile + Go Templates.

**Recuerda:**
- Port-forward es suficiente para desarrollo
- Ingress es útil para staging/producción
- Este capítulo (07) es completamente opcional

**💡 Tip final**: En producción real, usa Ingress con cert-manager para TLS automático. 
En desarrollo, port-forward es más que suficiente.