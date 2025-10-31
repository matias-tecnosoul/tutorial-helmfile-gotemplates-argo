# 07 - Ingress (45-60 min)

## 🎯 Objetivo

Exponer los microservices mediante Nginx Ingress Controller, con configuración dinámica por ambiente y testing de endpoints.

## 📝 ¿Qué es Ingress?

**Problema sin Ingress:**
```bash
# Cada service necesita LoadBalancer o NodePort
service/auth-service    LoadBalancer  35.xxx.xxx.xxx  # $$$
service/user-service    LoadBalancer  35.yyy.yyy.yyy  # $$$
service/api-gateway     LoadBalancer  35.zzz.zzz.zzz  # $$$
```

**Solución con Ingress:**
```bash
# Un solo LoadBalancer, múltiples rutas
ingress-nginx-controller  LoadBalancer  35.xxx.xxx.xxx

auth.example.com     → auth-service:80
users.example.com    → user-service:80
api.example.com      → api-gateway:80
```

## 🏗️ Nginx Ingress Controller

### helmfile.d/03-ingress.yaml (completo)

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
    wait: true
    timeout: 300
    labels:
      tier: networking
      component: ingress
    condition: ingressNginx.enabled
    needs:
      - {{ .Environment.Name }}/auth-service
      - {{ .Environment.Name }}/user-service
      - {{ .Environment.Name }}/api-gateway
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
  
  # Configuración del controller
  config:
    use-forwarded-headers: "true"
    compute-full-forwarded-for: "true"
    
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
  
  # Métricas (solo en prod)
  {{ if $isProd }}
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
  {{ end }}

# Default backend
defaultBackend:
  enabled: true
```

## 🌐 Ingress Resources en Charts

### charts/auth-service/templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  ingressClassName: nginx
  rules:
  {{- range .Values.ingress.hosts }}
  - host: {{ .host }}
    http:
      paths:
      {{- range .paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
        backend:
          service:
            name: {{ $.Release.Name }}
            port:
              number: 80
      {{- end }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
  {{- toYaml .Values.ingress.tls | nindent 2 }}
  {{- end }}
{{- end }}
```

### charts/auth-service/values.yaml (defaults)

```yaml
# ... (deployment, service config)

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: auth.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []
```

**Replica el mismo template para `user-service` y `api-gateway`.**

## 🎨 Values con Ingress

### helmfile.d/values/common.yaml (agregar)

```yaml
---
# Base domain
baseDomain: example.com

# Ingress
ingressNginx:
  enabled: true

# Services con ingress deshabilitado por defecto
services:
  authService:
    enabled: true
    ingress:
      enabled: false
      annotations: {}
  
  userService:
    enabled: true
    ingress:
      enabled: false
      annotations: {}
  
  apiGateway:
    enabled: true
    ingress:
      enabled: false
      annotations: {}
```

### helmfile.d/environments/dev/values.yaml (actualizar)

```yaml
---
baseDomain: dev.example.local

# Habilitar ingress en dev
services:
  authService:
    ingress:
      enabled: true
  
  userService:
    ingress:
      enabled: true
  
  apiGateway:
    ingress:
      enabled: true
```

### helmfile.d/environments/production/values.yaml

```yaml
---
baseDomain: example.com
clusterIssuer: letsencrypt-prod

services:
  authService:
    ingress:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        nginx.ingress.kubernetes.io/rate-limit: "100"
      tls:
        - secretName: auth-service-tls
          hosts:
            - auth.example.com
  
  userService:
    ingress:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      tls:
        - secretName: user-service-tls
          hosts:
            - users.example.com
  
  apiGateway:
    ingress:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        nginx.ingress.kubernetes.io/cors-allow-origin: "*"
      tls:
        - secretName: api-gateway-tls
          hosts:
            - api.example.com
```

### helmfile.d/values/auth-service/values.yaml.gotmpl (actualizar)

```yaml
---
{{ $env := .Environment.Name }}

# ... (imagen, replicas, env, resources)

# Ingress dinámico
{{ if .Values.services.authService.ingress.enabled }}
ingress:
  enabled: true
  className: nginx
  annotations:
    {{- toYaml .Values.services.authService.ingress.annotations | nindent 4 }}
  hosts:
    - host: auth.{{ .Values.baseDomain }}
      paths:
        - path: /
          pathType: Prefix
  {{ if .Values.services.authService.ingress.tls }}
  tls:
    {{- toYaml .Values.services.authService.ingress.tls | nindent 4 }}
  {{ end }}
{{ end }}
```

**Replica para `user-service` (host: users.xxx) y `api-gateway` (host: api.xxx).**

## 🚀 Deploy con Ingress

### Deploy completo

```bash
# Deploy todo (infraestructura → services → ingress)
helmfile -e dev apply

# Verificar ingress controller
kubectl get pods -n ingress-nginx

# Verificar ingress resources
kubectl get ingress -n dev
```

**Salida esperada:**
```
NAME             CLASS   HOSTS                    ADDRESS   PORTS   AGE
auth-service     nginx   auth.dev.example.local             80      1m
user-service     nginx   users.dev.example.local            80      1m
api-gateway      nginx   api.dev.example.local              80      1m
```

## 🧪 Testing de Endpoints

### Port-forward al Ingress Controller

```bash
# Terminal 1: Port-forward
kubectl port-forward -n ingress-nginx \
  svc/ingress-nginx-controller 8080:80
```

### Probar con curl (Host header)

```bash
# Terminal 2: Test endpoints

# Auth Service
curl -H "Host: auth.dev.example.local" http://localhost:8080/

# User Service
curl -H "Host: users.dev.example.local" http://localhost:8080/

# API Gateway
curl -H "Host: api.dev.example.local" http://localhost:8080/
```

**Salida esperada:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

### Script de testing automatizado

```bash
# scripts/test-endpoints.sh
#!/bin/bash

INGRESS_PORT=8080
DOMAIN="dev.example.local"

echo "🧪 Testing ingress endpoints..."
echo ""

test_endpoint() {
  local service=$1
  local host="${service}.${DOMAIN}"
  echo "Testing ${host}..."
  
  response=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Host: ${host}" \
    http://localhost:${INGRESS_PORT}/)
  
  if [ "$response" = "200" ]; then
    echo "✅ ${host} - OK (${response})"
  else
    echo "❌ ${host} - FAIL (${response})"
  fi
  echo ""
}

test_endpoint "auth"
test_endpoint "users"
test_endpoint "api"

echo "✅ Tests completed"
```

```bash
chmod +x scripts/test-endpoints.sh
./scripts/test-endpoints.sh
```

## 🌍 Hosts Locales (Desarrollo)

### Agregar a /etc/hosts

```bash
# /etc/hosts
127.0.0.1 auth.dev.example.local
127.0.0.1 users.dev.example.local
127.0.0.1 api.dev.example.local
```

### Probar con navegador

```bash
# Con port-forward activo
firefox http://auth.dev.example.local:8080
firefox http://users.dev.example.local:8080
firefox http://api.dev.example.local:8080
```

## 🎯 Configuraciones Avanzadas

### Rate limiting

```yaml
# production/values.yaml
services:
  apiGateway:
    ingress:
      annotations:
        nginx.ingress.kubernetes.io/rate-limit: "100"
        nginx.ingress.kubernetes.io/limit-rps: "10"
```

### CORS

```yaml
services:
  apiGateway:
    ingress:
      annotations:
        nginx.ingress.kubernetes.io/enable-cors: "true"
        nginx.ingress.kubernetes.io/cors-allow-origin: "*"
        nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
```

### Timeouts

```yaml
services:
  apiGateway:
    ingress:
      annotations:
        nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
        nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
        nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
```

### Path rewriting

```yaml
services:
  authService:
    ingress:
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /$2
      hosts:
        - host: api.example.com
          paths:
            - path: /auth(/|$)(.*)
              pathType: ImplementationSpecific
```

## 📊 Múltiples Paths en un Host

```yaml
# Todos los servicios bajo api.example.com
ingress:
  hosts:
    - host: api.example.com
      paths:
        - path: /auth
          pathType: Prefix
          backend:
            service:
              name: auth-service
        - path: /users
          pathType: Prefix
          backend:
            service:
              name: user-service
        - path: /
          pathType: Prefix
          backend:
            service:
              name: api-gateway
```

## 🐛 Troubleshooting

### Ingress no responde (404)

```bash
# Verificar ingress resource
kubectl describe ingress -n dev auth-service

# Verificar backend service
kubectl get svc -n dev auth-service

# Ver logs del controller
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

### Service backend no encontrado

```bash
# Error común: service name incorrecto
# Verificar que el service existe
kubectl get svc -n dev

# Verificar endpoints
kubectl get endpoints -n dev auth-service
```

### TLS/HTTPS no funciona

```bash
# Verificar secret existe (si usas TLS)
kubectl get secret -n dev auth-service-tls

# Ver certificado (requiere cert-manager)
kubectl describe certificate -n dev auth-service-tls
```

### Port-forward falla

```bash
# Error: unable to forward port

# Verificar que el pod está corriendo
kubectl get pods -n ingress-nginx

# Verificar service
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Usar puerto diferente si 8080 está ocupado
kubectl port-forward -n ingress-nginx \
  svc/ingress-nginx-controller 8888:80
```

## 📝 Ingress vs Service Types

| Tipo | Cuándo Usar | Costo | Ejemplo |
|------|-------------|-------|---------|
| **ClusterIP** | Interno solo | Gratis | DB, cache |
| **NodePort** | Dev/testing | Gratis | Kind clusters |
| **LoadBalancer** | Producción (sin ingress) | $$$ | 1 service = 1 LB |
| **Ingress** | Producción (múltiples apps) | $ | 1 LB para N apps |

## ✅ Checklist

- [ ] Desplegaste Nginx Ingress Controller
- [ ] Creaste Ingress resources para cada service
- [ ] Configuraste hosts dinámicos por ambiente
- [ ] Port-forward funciona y puedes hacer curl
- [ ] Script test-endpoints.sh ejecuta correctamente
- [ ] Entiendes annotations básicas (CORS, rate-limit)
- [ ] Hosts locales funcionan (opcional)

## ➡️ Siguiente Paso

👉 **[08 - Integración ArgoCD](08-integracion-argocd.md)**

Aprenderás:
- Instalar ArgoCD en tu cluster
- Conectar Helmfile con ArgoCD
- GitOps workflow completo
- Sync automático vs manual

---

**💡 Tip**: En Kind, usa NodePort (30080/30443). En clusters reales, usa LoadBalancer y configura DNS para apuntar a la IP del LB.