# 02 - Introducción a Helmfile (45 min)

## 🎯 Objetivo

Entender qué es Helmfile, por qué usarlo, y desplegar tu primer release (PostgreSQL) de forma declarativa.

## 📝 El Problema

Imagina gestionar 10+ aplicaciones en Kubernetes:

```bash
# Helm manual (repetitivo y propenso a errores)
helm install postgres bitnami/postgresql -n dev -f values-dev.yaml
helm install redis bitnami/redis -n dev -f values-dev.yaml
helm install auth-service ./charts/auth -n dev -f values-dev.yaml
helm install user-service ./charts/user -n dev -f values-dev.yaml
# ... 6 más

# Ahora hazlo en staging y production
# Ahora actualiza todos
# Ahora maneja dependencias
# Ahora... 🤯
```

**Problemas:**
- Comandos largos y repetitivos
- Difícil mantener consistencia entre ambientes
- No hay control de dependencias
- No hay state/versión de tu infraestructura
- Scripts bash frágiles

## 💡 La Solución: Helmfile

**Helmfile = Docker Compose para Helm**

Un archivo declarativo que define TODOS tus releases:

```yaml
# helmfile.yaml
releases:
  - name: postgres
    chart: groundhog2k/postgres
    namespace: dev
    values:
      - values/postgres.yaml
  
  - name: redis
    chart: groundhog2k/redis
    namespace: dev
    values:
      - values/redis.yaml
```

Un comando para gobernarlos a todos:
```bash
helmfile apply  # Instala/actualiza TODO
```

## 🏗️ Tu Primer Helmfile

### Paso 1: Estructura básica

```bash
cd ~/tutorial-helmfile-gotemplates-argo

# Crear estructura inicial
mkdir -p helmfile.d/{values,environments}
mkdir -p helmfile.d/values/postgres
```

### Paso 2: helmfile.yaml (raíz)

```yaml
# helmfile.yaml
---
# Definir ambientes
environments:
  dev:
    kubeContext: kind-helmfile-tutorial
    values:
      - helmfile.d/values/common.yaml

---
# Orquestador simple (por ahora)
helmfiles:
  - path: helmfile.d/01-infrastructure.yaml
```

### Paso 3: Values comunes

```yaml
# helmfile.d/values/common.yaml
---
# Valores base compartidos

postgres:
  enabled: true
  image:
    repository: postgres
    tag: "15-alpine"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  storage:
    size: 1Gi
```

### Paso 4: Helmfile de infraestructura

```yaml
# helmfile.d/01-infrastructure.yaml
---
# Heredar configuración de ambientes
environments:
  dev:
    kubeContext: kind-helmfile-tutorial
    values:
      - values/common.yaml

---
# Repositorios de Helm
repositories:
  - name: groundhog2k
    url: https://groundhog2k.github.io/helm-charts/

---
# Releases
releases:
  - name: postgres
    namespace: {{ .Environment.Name }}
    createNamespace: true
    chart: groundhog2k/postgres
    version: ~0.7.0
    values:
      - values/postgres/values.yaml
    labels:
      tier: infrastructure
      component: database
    condition: postgres.enabled
```

**Nota:** `{{ .Environment.Name }}` es Go Template (lo veremos en cap 03).

### Paso 5: Values específicos de PostgreSQL

```yaml
# helmfile.d/values/postgres/values.yaml
---
image:
  repository: postgres
  tag: "15-alpine"

env:
  - name: POSTGRES_DB
    value: appdb
  - name: POSTGRES_USER
    value: appuser
  - name: POSTGRES_PASSWORD
    value: dev-password-123

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

storage:
  className: standard
  requestedSize: 1Gi
```

## 🧪 Desplegar con Helmfile

### Listar releases (sin instalar)

```bash
helmfile -e dev list
```

**Salida:**
```
NAME     NAMESPACE  ENABLED  LABELS                              CHART                       VERSION
postgres dev        true     component:database,tier:infrastructure  groundhog2k/postgres  ~0.7.0
```

### Ver qué se instalará (dry-run)

```bash
helmfile -e dev diff
```

**Primera vez:** Mostrará que todo es nuevo.

### Instalar

```bash
helmfile -e dev apply
```

**Salida:**
```
Adding repo groundhog2k https://groundhog2k.github.io/helm-charts/
"groundhog2k" has been added to your repositories

Comparing release=postgres, chart=groundhog2k/postgres
postgres, default, Deployment (apps) has been added:
...

Upgrading release=postgres, chart=groundhog2k/postgres
Release "postgres" does not exist. Installing it now.
NAME: postgres
...
STATUS: deployed
```

### Verificar

```bash
# Ver releases con Helm
helm list -n dev

# Ver recursos en Kubernetes
kubectl get all -n dev

# Ver logs del pod
kubectl logs -n dev -l app.kubernetes.io/name=postgres
```

**Salida esperada:**
```
NAME                             READY   STATUS    RESTARTS   AGE
pod/postgres-0                   1/1     Running   0          30s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/postgres   ClusterIP   10.96.100.123   <none>        5432/TCP   30s

NAME                        READY   AGE
statefulset.apps/postgres   1/1     30s
```

### Conectarse a PostgreSQL

```bash
# Port-forward
kubectl port-forward -n dev svc/postgres 5432:5432 &

# Conectar con psql (si lo tienes instalado)
PGPASSWORD=dev-password-123 psql -h localhost -U appuser -d appdb

# O desde un pod temporal
kubectl run -it --rm psql --image=postgres:15-alpine --restart=Never -- \
  psql -h postgres.dev.svc.cluster.local -U appuser -d appdb
```

## 🔄 Actualizar un Release

### Cambiar configuración

Edita `helmfile.d/values/postgres/values.yaml`:

```yaml
# Cambiar límite de memoria
resources:
  limits:
    cpu: 500m
    memory: 1Gi  # ← Cambio: de 512Mi a 1Gi
```

### Ver diferencias

```bash
helmfile -e dev diff
```

**Salida:**
```diff
postgres, Deployment (apps) has changed:
  spec:
    template:
      spec:
        containers:
        - resources:
            limits:
-             memory: 512Mi
+             memory: 1Gi
```

### Aplicar cambios

```bash
helmfile -e dev apply
```

Helmfile detecta el cambio y hace upgrade automáticamente.

## 🗑️ Eliminar un Release

```bash
# Eliminar todo
helmfile -e dev destroy

# Confirmar
kubectl get all -n dev  # No debería haber nada
```

## 📊 Comparación: Helm vs Helmfile

| Aspecto | Helm Manual | Helmfile |
|---------|-------------|----------|
| **Instalar** | `helm install postgres groundhog2k/postgres -n dev -f values.yaml` | `helmfile apply` |
| **Actualizar** | `helm upgrade postgres groundhog2k/postgres -n dev -f values.yaml` | `helmfile apply` |
| **Ver cambios** | `helm diff upgrade postgres ...` (requiere plugin) | `helmfile diff` |
| **Múltiples apps** | Script bash con loops | Declarativo en YAML |
| **Ambientes** | Archivos values separados + scripts | `helmfile -e prod apply` |
| **Dependencias** | Manual o Helm hooks | `needs:` built-in |
| **State** | Solo en cluster | Archivo versionable en Git |

## 🎯 Comandos Clave de Helmfile

```bash
# Listar releases
helmfile list
helmfile -e dev list

# Ver diferencias (antes de aplicar)
helmfile diff
helmfile -e dev diff

# Aplicar cambios (instalar o actualizar)
helmfile apply
helmfile -e dev apply

# Solo sincronizar (sin preguntar)
helmfile sync

# Eliminar todo
helmfile destroy
helmfile -e dev destroy

# Ver templates generados (sin aplicar)
helmfile template
helmfile -e dev template

# Aplicar solo un release específico
helmfile -e dev -l name=postgres apply

# Aplicar por labels
helmfile -e dev -l tier=infrastructure apply
```

## 🔍 Estructura del helmfile.yaml

```yaml
# 1. Ambientes (opcional)
environments:
  dev:
    values: [...]
  production:
    values: [...]

# 2. Repositorios
repositories:
  - name: repo-name
    url: https://...

# 3. Releases
releases:
  - name: release-name
    namespace: namespace
    chart: repo/chart
    version: x.y.z
    values:
      - path/to/values.yaml
    condition: enabled.flag  # Opcional
    needs: [other-release]    # Opcional
    labels:                   # Opcional
      key: value
```

## ✅ Checklist

Antes de continuar:

- [ ] Entiendes qué problema resuelve Helmfile
- [ ] Creaste helmfile.yaml y helmfile.d/01-infrastructure.yaml
- [ ] Desplegaste PostgreSQL con `helmfile apply`
- [ ] Verificaste con `kubectl get all -n dev`
- [ ] Probaste `helmfile diff` y viste cambios
- [ ] Ejecutaste `helmfile list` exitosamente

## ➡️ Siguiente Paso

👉 **[03 - Go Templates](03-go-templates.md)**

En el próximo capítulo profundizaremos en:
- Variables y asignación
- Condicionales (if/else)
- Loops (range)
- Pipelines y funciones
- Templating de values dinámicos

---

**💡 Tip**: Guarda el comando `helmfile diff` antes de `apply`. Es tu mejor amigo para evitar sorpresas.