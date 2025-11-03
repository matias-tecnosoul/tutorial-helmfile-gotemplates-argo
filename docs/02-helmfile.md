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
# helmfile.d/01-infrastructure.yaml
releases:
  - name: postgres
    chart: groundhog2k/postgres
    namespace: dev
    values:
      - values/postgres/values.yaml.gotmpl
  
  - name: redis
    chart: groundhog2k/redis
    namespace: dev
    values:
      - values/redis/values.yaml.gotmpl
```

Un comando para gobernarlos a todos:
```bash
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
```

> 💡 **Diferencia con otros proyectos**:
> 
> Este tutorial sigue el patrón de Mikroways (repo `k8s-base-services`) donde NO hay 
> `helmfile.yaml` en la raíz. Cada módulo se ejecuta independientemente:
> 
> **Ventajas:**
> - Deploy selectivo por módulo (solo infra, solo services, etc.)
> - Más control granular
> - Patrón usado en producción

## 🗂️ Tu Primer Helmfile

### Paso 1: Estructura básica

```bash
cd ~/tutorial-helmfile-gotemplates-argo

# Crear estructura inicial
mkdir -p helmfile.d/{values,environments}
mkdir -p helmfile.d/values/postgres
mkdir -p helmfile.d/environments/dev
```

### Paso 2: Values comunes

```yaml
# helmfile.d/values/common.yaml
---
# Valores base compartidos

postgres:
  enabled: true
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
```

### Paso 3: Secrets por ambiente

```yaml
# helmfile.d/environments/dev/secrets.yaml
---
postgres:
  password: "dev-password-123"
```

> ⚠️ **IMPORTANTE**: En producción real, estos secrets deben estar cifrados con SOPS.
> Para este tutorial usamos plain text para simplificar.

### Paso 4: Helmfile de infraestructura

```yaml
# helmfile.d/01-infrastructure.yaml
---
# Configuración de ambientes
environments:
  dev:
    kubeContext: kind-helmfile-tutorial
    values:
      - values/common.yaml
      - environments/dev/values.yaml
      - environments/dev/secrets.yaml

---
# Repositorios de Helm
repositories:
  - name: groundhog2k
    url: https://groundhog2k.github.io/helm-charts/

---
# Releases
releases:
  - name: postgres
    namespace: dev
    createNamespace: true
    chart: groundhog2k/postgres
    version: ~1.5.0
    values:
      - values/postgres/values.yaml.gotmpl
    labels:
      tier: infrastructure
      component: database
    condition: postgres.enabled
```

> 💡 **Sobre versiones de charts**:
> 
> El prefijo `~` permite updates de parche:
> - `~1.5.0` → acepta `1.5.x` (1.5.11, 1.5.12, etc.)
> - `~1.0` → acepta `1.x` (1.5.0, 1.6.0, etc.)
> 
> Para ver versiones disponibles:
> ```bash
> helm search repo groundhog2k/postgres --versions
> ```

### Paso 5: Values específicos de PostgreSQL (con templates)

```yaml
# helmfile.d/values/postgres/values.yaml.gotmpl
---
image:
  repository: {{ .Values.postgres.image.repository }}
  tag: {{ .Values.postgres.image.tag | quote }}

env:
  - name: POSTGRES_DB
    value: {{ index .Values.postgres.databases 0 | quote }}
  
  - name: POSTGRES_USER
    value: "appuser"
  
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
```

> 💡 **¿Por qué `.gotmpl`?**
> 
> Los archivos en `values/servicio/` necesitan `.gotmpl` cuando usan Go Templates (`{{ }}`):
> 
> - ✅ `values/postgres/values.yaml.gotmpl` - Usa `{{ .Values.* }}`
> - ❌ `values/common.yaml` - Solo valores estáticos (sin .gotmpl)
> - ✅ `01-infrastructure.yaml` - Puede usar templates pero NO necesita .gotmpl
> 
> **Regla:** Solo los values files que referencian `.Values.*` necesitan `.gotmpl`

## 🧪 Desplegar con Helmfile

### Listar releases (sin instalar)

```bash
helmfile -f helmfile.d/01-infrastructure.yaml -e dev list
```

**Salida:**
```
NAME     NAMESPACE  ENABLED  LABELS                              CHART                 VERSION
postgres dev        true     component:database,tier:infrastructure  groundhog2k/postgres  ~1.5.0
```

### Ver qué se instalará (dry-run)

```bash
helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff
```

**Primera vez:** Mostrará que todo es nuevo.

### Instalar

```bash
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
```

**Salida:**
```
Adding repo groundhog2k https://groundhog2k.github.io/helm-charts/
"groundhog2k" has been added to your repositories

Comparing release=postgres, chart=groundhog2k/postgres
postgres, Service (v1) has been added:
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
NAME         READY   STATUS    RESTARTS   AGE
postgres-0   1/1     Running   0          30s

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

Edita `helmfile.d/values/common.yaml`:

```yaml
# Cambiar límite de memoria
resources:
  limits:
    cpu: 500m
    memory: 1Gi  # ← Cambio: de 512Mi a 1Gi
```

### Ver diferencias

```bash
helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff
```

**Salida:**
```diff
postgres, StatefulSet (apps) has changed:
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
helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply
```

Helmfile detecta el cambio y hace upgrade automáticamente.

## 🗑️ Eliminar un Release

```bash
# Eliminar todo
helmfile -f helmfile.d/01-infrastructure.yaml -e dev destroy

# Confirmar
kubectl get all -n dev  # No debería haber nada
```

## 📊 Comparación: Helm vs Helmfile

| Aspecto | Helm Manual | Helmfile |
|---------|-------------|----------|
| **Instalar** | `helm install postgres groundhog2k/postgres -n dev -f values.yaml` | `helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply` |
| **Actualizar** | `helm upgrade postgres groundhog2k/postgres -n dev -f values.yaml` | `helmfile -f helmfile.d/01-infrastructure.yaml -e dev apply` |
| **Ver cambios** | `helm diff upgrade postgres ...` (requiere plugin) | `helmfile -f helmfile.d/01-infrastructure.yaml -e dev diff` |
| **Múltiples apps** | Script bash con loops | Declarativo en YAML |
| **Ambientes** | Archivos values separados + scripts | Cambiar `-e dev` por `-e prod` |
| **Dependencias** | Manual o Helm hooks | `needs:` built-in |
| **State** | Solo en cluster | Archivo versionable en Git |

## 🎯 Comandos Clave de Helmfile

```bash
# Alias útil (opcional)
alias hf-infra='helmfile -f helmfile.d/01-infrastructure.yaml'

# Listar releases
hf-infra -e dev list

# Ver diferencias (antes de aplicar)
hf-infra -e dev diff

# Aplicar cambios (instalar o actualizar)
hf-infra -e dev apply

# Solo sincronizar (sin preguntar)
hf-infra -e dev sync

# Eliminar todo
hf-infra -e dev destroy

# Ver templates generados (sin aplicar)
hf-infra -e dev template

# Aplicar solo un release específico
hf-infra -e dev -l name=postgres apply

# Aplicar por labels
hf-infra -e dev -l tier=infrastructure apply
```

## 🔍 Estructura del helmfile modular

```yaml
# helmfile.d/01-infrastructure.yaml

# 1. Ambientes
environments:
  dev:
    kubeContext: kind-helmfile-tutorial
    values:
      - values/common.yaml
      - environments/dev/values.yaml
      - environments/dev/secrets.yaml

# 2. Repositorios
repositories:
  - name: repo-name
    url: https://...

# 3. Releases
releases:
  - name: release-name
    namespace: dev
    chart: repo/chart
    version: x.y.z
    values:
      - values/servicio/values.yaml.gotmpl
    condition: enabled.flag  # Opcional
    labels:                  # Opcional
      key: value
```

## 🔄 Flujo de carga de valores

Helmfile carga valores en orden y los mergea:

```
common.yaml → environments/dev/values.yaml → secrets.yaml
                           ↓
                Valores mergeados disponibles en .Values
                           ↓
            values/postgres/values.yaml.gotmpl
                 accede a {{ .Values.* }}
```

**Ejemplo:**

```yaml
# common.yaml
postgres:
  image:
    tag: "15-alpine"
  
# environments/dev/values.yaml
postgres:
  image:
    tag: "16-alpine"  # ← Override

# values/postgres/values.yaml.gotmpl
image:
  tag: {{ .Values.postgres.image.tag }}  # ← Resultado: "16-alpine"
```

> 💡 **Orden importa**: El último archivo gana en conflictos.

## ✅ Checklist

Antes de continuar:

- [ ] Entiendes qué problema resuelve Helmfile
- [ ] Creaste la estructura `helmfile.d/` con valores y environments
- [ ] Desplegaste PostgreSQL con `helmfile -f ... apply`
- [ ] Verificaste con `kubectl get all -n dev`
- [ ] Probaste `helmfile ... diff` y viste cambios
- [ ] Ejecutaste `helmfile ... list` exitosamente
- [ ] Entiendes el flujo de carga de valores (common → env → secrets)

## ➡️ Siguiente Paso

👉 **[03 - Go Templates](03-go-templates.md)**

En el próximo capítulo profundizaremos en:
- Variables y asignación
- Condicionales (if/else)
- Loops (range)
- Pipelines y funciones
- Templating de values dinámicos

---

**💡 Tip**: Ejecuta `helmfile ... diff` antes de cada `apply`. Es tu mejor amigo para evitar sorpresas.