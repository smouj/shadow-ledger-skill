---
name: Shadow Ledger
category: audit
purpose: Audit and traceability for operations
tags: [audit, traceability, logging, compliance, operations]
version: 1.0.0
requires:
  - openclaw-cli >= 2.0.0
  - persistent storage (SQLite or PostgreSQL)
  - file integrity tools (sha256sum, gpg)
---

# Shadow Ledger

**Shadow Ledger** es una skill de auditoría y trazabilidad para OpenClaw que proporciona un registro inmutable y verificable de todas las operaciones ejecutadas en el sistema. Diseñado para cumplimiento normativo, investigación de incidentes y accountability operativo.

## Uso Primario

```bash
# Registrar una operación manual
openclaw ledger record --operation "deploy-production" --actor "devops@team" --target "rpgclaw:v2.5.0" --status "success" --meta '{"environment":"production","branch":"main"}'

# Verificar integridad de un registro
openclaw ledger verify --id "sl-20260301-00042"

# Generar reporte de auditoría
openclaw ledger report --from "2026-01-01" --to "2026-03-01" --format "markdown"

# Listar operaciones críticas
openclaw ledger query --severity "critical" --limit 50
```

## Casos de Uso Reales

### Caso 1: Auditoría Post-Incidente

Un usuario reporta acceso no autorizado a datos sensibles. Necesitas reconstruir la cadena de eventos:

```bash
# 1. Buscar todas las operaciones en el rango temporal del incidente
openclaw ledger query --from "2026-02-28T14:00:00Z" --to "2026-02-28T18:00:00Z" --actor "unknown" --format "json"

# 2. Verificar que los registros no fueron alterados (hash chain integrity)
openclaw ledger integrity-check --start "2026-02-28" --end "2026-02-28"

# 3. Exportar evidencia para investigación forense
openclaw ledger export --query 'actor:* AND target:database*' --output "/tmp/incident-evidence-20260228.json" --sign
```

### Caso 2: Cumplimiento GDPR/PCI-DSS

Demostrar que los datos de usuarios fueron procesados según políticas establecidas:

```bash
# 1. Registrar acceso a datos de usuario
openclaw ledger record \
  --operation "user-data-export" \
  --actor "compliance-officer@company" \
  --target "user-db:customers" \
  --status "success" \
  --meta '{"purpose":" quarterly-audit","retention-days":90,"legal-basis":"legitimate-interest"}'

# 2. Generar reporte de cumplimiento para período específico
openclaw ledger compliance-report \
  --standard "GDPR-ART-30" \
  --from "2026-01-01" \
  --to "2026-03-01" \
  --output "gdpr-compliance-q1-2026.pdf"

# 3. Listar todos los accesos a datos personales
openclaw ledger query --operation "*-data-*" --format "table" --columns "timestamp,actor,operation,target"
```

### Caso 3: Cambio de Infraestructura VPS

Auditar cambios críticos en la infraestructura sin confirmación explícita:

```bash
# 1. Antes de ejecutar cambios sensibles, verificar historial reciente
openclaw ledger query --target "vps-prod*" --severity "high" --hours 24

# 2. Registrar el cambio planeado ANTES de ejecutarlo
openclaw ledger record \
  --operation "infrastructure-change" \
  --actor "vps-ops" \
  --target "vps-prod-01:firewall-rules" \
  --status "pending" \
  --meta '{"action":"add-rule","port":443,"source":"0.0.0.0/0","reason":"migration"}' \
  --ttl 86400

# 3. Después de ejecutar, actualizar el estado
openclaw ledger update --id "sl-$(date +%Y%m%d)-*" --status "success" --meta '{"executed-at":"2026-03-01T10:30:00Z"}'

# 4. Verificar que el cambio fue registrado correctamente
openclaw ledger verify --id "sl-20260301-*"
```

### Caso 4: Revisión de Permisos de Acceso

Auditar quién tuvo acceso a recursos críticos en el último mes:

```bash
# 1. Listar todos los accesos a recursos protegidos
openclaw ledger query \
  --target "prod/*" \
  --operation "access-*" \
  --from "2026-02-01" \
  --format "csv" \
  --output "access-audit-feb2026.csv"

# 2. Identificar accesos fuera de horario laboral (sospechosos)
openclaw ledger query \
  --target "prod/*" \
  --from "2026-02-01" \
  --format "json" | jq '.[] | select(.timestamp | strptime("%Y-%m-%dT%H:%M:%SZ") | .tm_hour < 7 or .tm_hour > 22)'

# 3. Generar resumen ejecutivo para management
openclaw ledger summary --from "2026-02-01" --to "2026-02-28" --group-by "actor"
```

## Comandos Detallados

### `openclaw ledger init`

Inicializa la base de datos de auditoría. Debe ejecutarse una sola vez por workspace.

```bash
# Inicializar con SQLite (por defecto)
openclaw ledger init

# Inicializar con PostgreSQL (para producción)
openclaw ledger init --backend "postgres" --connection "postgresql://user:pass@localhost:5432/audit_db"
```

**Opciones:**
- `--backend`: `sqlite` (default) o `postgres`
- `--connection`: URL de conexión PostgreSQL
- `--encryption`: Habilitar cifrado en reposo (`aes-256-gcm`)
- `--retention-days`: Días de retención (default: 365)

### `openclaw ledger record`

Registra una nueva operación en el ledger.

```bash
# Registro básico
openclaw ledger record --operation "deploy" --actor "ci-bot" --target "app:v1.2.3" --status "success"

# Registro con metadatos y TTL
openclaw ledger record \
  --operation "config-change" \
  --actor "admin@company.com" \
  --target "app:database-pool" \
  --status "success" \
  --meta '{"old-value":10,"new-value":20,"reason":"load-testing"}' \
  --ttl 2592000 \
  --severity "medium"

# Registro con firma digital (non-repudiation)
openclaw ledger record \
  --operation "approve-expense" \
  --actor "cfo@company.com" \
  --target "expense:#4521" \
  --status "success" \
  --sign
```

**Opciones:**
- `--operation`: Nombre de la operación (requerido)
- `--actor`: Identificador del actor (requerido)
- `--target`: Recurso o entidad afectada (requerido)
- `--status`: `success`, `failure`, `pending`, `cancelled`
- `--severity`: `low`, `medium`, `high`, `critical`
- `--meta`: JSON con metadatos adicionales
- `--ttl`: Segundos hasta que el registro expire (0 = nunca)
- `--sign`: Firmar digitalmente el registro
- `--hash`: Hash SHA-256 de un archivo relacionado

### `openclaw ledger query`

Consulta registros del ledger con filtros avanzados.

```bash
# Query básico por actor
openclaw ledger query --actor "devops@team"

# Query con rango temporal
openclaw ledger query --from "2026-02-01" --to "2026-02-28" --operation "deploy*"

# Query con expresión regular
openclaw ledger query --target "prod/db/*" --severity "critical" --limit 100

# Query con formato específico
openclaw ledger query --actor "*@company.com" --format "json" --pretty

# Query con filtros combinados
openclaw ledger query \
  --from "2026-01-01" \
  --operation "delete*" \
  --status "success" \
  --severity "high" \
  --format "table"
```

**Opciones:**
- `--from`, `--to`: Rango temporal (ISO 8601)
- `--actor`: Filtrar por actor (soporta wildcards `*`)
- `--operation`: Filtrar por tipo de operación
- `--target`: Filtrar por recurso objetivo
- `--status`: Filtrar por estado
- `--severity`: Filtrar por severidad
- `--limit`: Límite de resultados (default: 50)
- `--format`: `table`, `json`, `csv`, `markdown`
- `--pretty`: Formateo JSON legible

### `openclaw ledger verify`

Verifica la integridad criptográfica de uno o más registros.

```bash
# Verificar un registro específico
openclaw ledger verify --id "sl-20260301-00042"

# Verificar cadena de integridad de un día completo
openclaw ledger verify --date "2026-02-28"

# Verificar todos los registros y generar reporte
openclaw ledger verify --full --output "integrity-report.txt"
```

**Opciones:**
- `--id`: ID del registro a verificar
- `--date`: Verificar todos los registros de una fecha
- `--full`: Verificación completa incluyendo firma GPG
- `--output`: Archivo de salida para el reporte

### `openclaw ledger report`

Genera reportes de auditoría en múltiples formatos.

```bash
# Reporte de actividad mensual
openclaw ledger report --from "2026-02-01" --to "2026-02-28" --format "markdown" --output "audit-feb2026.md"

# Reporte en PDF para gestión
openclaw ledger report --from "2026-01-01" --to "2026-03-01" --format "pdf" --template "executive"

# Reporte CSV para análisis
openclaw ledger report --operation "deploy*" --format "csv" --output "deploys.csv"

# Reporte de compliance específico
openclaw ledger report --standard "SOC2-Type2" --from "2026-01-01" --output "soc2-audit.log"
```

**Opciones:**
- `--from`, `--to`: Período del reporte
- `--format`: `markdown`, `pdf`, `csv`, `json`, `html`
- `--template`: Plantilla de reporte (`executive`, `detailed`, `compliance`)
- `--standard`: Estándar de compliance (`GDPR`, `PCI-DSS`, `SOC2`, `HIPAA`)
- `--group-by`: Agrupar resultados (`actor`, `operation`, `day`, `severity`)

### `openclaw ledger alert`

Configura alertas basadas en patrones de actividad sospechosas.

```bash
# Alertar si más de 5 failed logins en 10 minutos
openclaw ledger alert add \
  --name "excessive-failures" \
  --query 'status:failure AND operation:login*' \
  --threshold 5 \
  --window 600 \
  --action "notify" \
  --channels "slack:#security,email:sec@company.com"

# Alertar si acceso a datos críticos fuera de horario
openclaw ledger alert add \
  --name "off-hours-access" \
  --query 'target:prod/database/* AND severity:critical' \
  --condition "hour < 7 OR hour > 22" \
  --action "pagerduty"

# Listar alertas activas
openclaw ledger alert list

# Desactivar alerta
openclaw ledger alert remove "excessive-failures"
```

## Formato de Registro Interno

Cada entrada en el ledger tiene la siguiente estructura:

```json
{
  "id": "sl-20260301-00042",
  "timestamp": "2026-03-01T10:30:00.123Z",
  "operation": "deploy",
  "actor": "ci-bot",
  "actor_ip": "10.0.1.50",
  "target": "rpgclaw:v2.5.0",
  "status": "success",
  "severity": "high",
  "meta": {
    "environment": "production",
    "branch": "main",
    "commit": "a1b2c3d",
    "runner": "github-actions"
  },
  "hash": "sha256:3a4b5c...",
  "previous_hash": "sl-20260301-00041",
  "signature": "gpg:ABCD1234...",
  "expires_at": null
}
```

## Troubleshooting

### Problema: "Ledger not initialized"

**Síntoma:** Cualquier comando retorna `Error: Ledger database not initialized`

**Solución:**
```bash
# Verificar si existe la base de datos
ls -la ~/.openclaw/ledger.db

# Si no existe, inicializar
openclaw ledger init

# Si existe pero hay permisos incorrectos
chmod 600 ~/.openclaw/ledger.db
chown $USER ~/.openclaw/ledger.db
```

### Problema: "Integrity check failed"

**Síntoma:** `openclaw ledger verify` reporta hash mismatch

**Causa probable:** El registro fue modificado después de su creación

**Solución:**
```bash
# 1. Identificar registros corruptos
openclaw ledger verify --full 2>&1 | grep "MISMATCH"

# 2. Verificar si hay backup reciente
ls -la ~/.openclaw/backups/ledger-*.db

# 3. Restaurar desde backup si es necesario
cp ~/.openclaw/backups/ledger-20260228.db ~/.openclaw/ledger.db

# 4. Investigar el incidente (quién/modificó qué)
openclaw ledger query --id "sl-*" --from "2026-02-28" --format "json" | jq '.[] | select(.modified != null)'
```

### Problema: "Query timeout on large datasets"

**Síntoma:** Queries lentos o timeout en tablas con >1M registros

**Solución:**
```bash
# 1. Añadir índices manualmente si no existen
openclaw ledger optimize

# 2. Usar filtros más específicos (limitar rango temporal)
openclaw ledger query --from "2026-02-01" --to "2026-02-02" --target "app:*"

# 3. Exportar a archivo y procesar externamente
openclaw ledger query --from "2026-01-01" --format "json" --output "large query.json"

# 4. Considerar migrar a PostgreSQL para mejor rendimiento
openclaw ledger migrate --backend postgres --connection "postgresql://..."
```

### Problema: "Signature verification failed"

**Síntoma:** No se puede verificar la firma digital de un registro

**Solución:**
```bash
# 1. Verificar que la clave GPG esté disponible
gpg --list-keys "OpenClaw Ledger"

# 2. Si no existe, regenerar clave
openclaw ledger gpg-reinit

# 3. Verificar manualmente
gpg --verify <(echo "$RECORD_DATA") "$SIGNATURE"
```

### Problema: "Retention policy not working"

**Síntoma:** Registros antiguos no se borran automáticamente

**Solución:**
```bash
# 1. Verificar configuración de retención
openclaw ledger config | grep retention

# 2. Forzar limpieza manual
openclaw ledger purge --dry-run
openclaw ledger purge --confirm

# 3. Configurar cron para limpieza automática
# Añadir a crontab:
# 0 3 * * * openclaw ledger purge --confirm >> /var/log/ledger-purge.log 2>&1
```

## Buenas Prácticas

1. **Registrar ANTES de ejecutar operaciones sensibles** - Usa `--status pending` y actualiza después
2. **Incluir contexto suficiente en `--meta`** - Facilita investigación posterior
3. **Firmar digitalmente** registros críticos - Úsalo para compliance
4. **Rotar claves GPG** anualmente - Mantiene seguridad a largo plazo
5. **Monitorear alertas** - Configura notificaciones para patrones sospechosos
6. **Backups regulares** - El ledger es evidencia; protege su integridad
7. **No almacenar datos sensibles** en `--meta` - Hash en su lugar

## Integración con Otras Skills

### Con VPS-OPS

```bash
# En script de deployment, añadir:
openclaw ledger record \
  --operation "vps-deploy" \
  --actor "vps-ops" \
  --target "vps:${SERVER_NAME}" \
  --status "success" \
  --meta "$(cat deployment-meta.json)"
```

### Con RPGCLAW-OPS

```bash
# Hook en CI/CD para registrar builds
openclaw ledger record \
  --operation "ci-build" \
  --actor "github-actions" \
  --target "rpgclaw:${GITHUB_SHA}" \
  --status "success" \
  --meta "{\"branch\":\"${GITHUB_REF}\",\"workflow\":\"${GITHUB_WORKFLOW}\"}"
```

## Ejemplo Completo de Flujo

```bash
#!/bin/bash
# Ejemplo: Flujo completo de auditoría para deploy

set -e

TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)
ENVIRONMENT="production"
VERSION="$1"

echo "=== Shadow Ledger Audit Trail ==="

# 1. Pre-deploy: Registrar intención
openclaw ledger record \
  --operation "deploy-start" \
  --actor "ci-bot" \
  --target "rpgclaw:${VERSION}" \
  --status "pending" \
  --severity "high" \
  --meta "{\"environment\":\"${ENVIRONMENT}\",\"started-at\":\"${TIMESTAMP}\"}" \
  --sign

LEDGER_ID=$(openclaw ledger query --operation "deploy-start" --target "rpgclaw:${VERSION}" --limit 1 --format "json" | jq -r '.[0].id')

# 2. Ejecutar deployment (ejemplo con docker)
docker-compose -f production.yml up -d --build

# 3. Post-deploy: Verificar salud
HEALTH=$(curl -sf http://localhost:3000/api/health || echo "failed")

# 4. Actualizar registro con resultado
if [ "$HEALTH" = "healthy" ]; then
  openclaw ledger update \
    --id "$LEDGER_ID" \
    --status "success" \
    --meta "{\"health-check\":\"passed\",\"completed-at\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}"
else
  openclaw ledger update \
    --id "$LEDGER_ID" \
    --status "failure" \
    --meta "{\"health-check\":\"failed\",\"error\":\"service-unhealthy\",\"completed-at\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}"
  exit 1
fi

# 5. Verificar integridad del registro
openclaw ledger verify --id "$LEDGER_ID"

echo "=== Deployment audit complete ==="
```

---

**Nota:** Esta skill requiere que la base de datos del ledger esté inicializada antes de su uso. Ejecuta `openclaw ledger init` como primer paso.
```