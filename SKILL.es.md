```markdown
---
name: Logic Weaver
slug: logic-weaver
version: 1.0.0
description: Teje flujos de lógica intrincados en aplicaciones usando máquinas de estados, árboles de decisión y patrones de flujo de trabajo
author: OpenClaw Team
tags: [logic, flow, architecture, state-machine, workflow]
dependencies:
  - nodejs: ">=18.0.0"
  - typescript: ">=5.0.0"
  - xstate: "^5.0.0"
  - commander: "^11.0.0"
  - chalk: "^4.0.0"
env_vars:
  - LOGIC_WEAVER_DEBUG
  - LOGIC_WEAVER_CONFIG_PATH
entrypoint: logic-weaver
---

# Logic Weaver

## Propósito

Logic Weaver crea, gestiona y despliega flujos de lógica complejos para aplicaciones usando configuraciones YAML declarativas. Maneja máquinas de estados, árboles de decisión y flujos de trabajo de múltiples pasos con validación, pruebas y visualización integradas.

### Casos de uso REALES:

1. **Procesamiento de pedidos de comercio electrónico**: Teje lógica para validación de carrito → pago → inventario → envío → notificaciones
2. **Flujos de incorporación de usuarios**: Registro multifuncional con ramas condicionales basadas en tipo de usuario y datos
3. **Árboles de decisión de IA de juegos**: Comportamiento NPC complejo con persistencia y transiciones de estado
4. **Orquestación de pipelines CI/CD**: Definir flujos de despliegue con activadores de rollback
5. **Asistentes de formulario**: Formularios multipágina con lógica de omisión, validación de puertas y ramificación dinámica

## Alcance

### Comandos:

```bash
# Crear un nuevo flujo de lógica desde plantilla
logic-weaver init <flow-name> --template <state-machine|decision-tree|workflow>

# Definir nodos y transiciones en YAML
logic-weaver design --file flows/order-processing.yaml

# Validar integridad del flujo de lógica
logic-weaver validate --file flows/order-processing.yaml --strict

# Generar implementación TypeScript/JavaScript
logic-weaver generate --file flows/order-processing.yaml --lang ts --output src/flows/

# Visualizar flujo como SVG/PNG
logic-weaver visualize --file flows/order-processing.yaml --format svg --output docs/flow-diagram.svg

# Probar flujo con entradas de simulación
logic-weaver test --file flows/order-processing.yaml --scenario tests/order-success.json

# Desplegar flujo en tiempo de ejecución (servicio XState, runtime personalizado)
logic-weaver deploy --file flows/order-processing.yaml --runtime xstate --endpoint /api/flows/order

# Comparar dos versiones de flujo
logic-weaver diff --file v1.yaml v2.yaml --format table

# Listar todos los flujos con estado
logic-weaver list --environment production

# Ejecutar paso único manualmente (debug)
logic-weaver step --flow order-processing --state PaymentFailed --input '{\"orderId\":\"123\"}'
```

### Patrones de Archivos:

- Definiciones de flujo: `**/flows/*.yaml` o `**/logic/*.yaml`
- Escenarios de prueba: `**/tests/**/*.json` (emparejados con nombre de flujo)
- Código generado: `src/flows/generated/`
- Config: `.logic-weaverrc.json` o `logic-weaver.config.yaml`

## Proceso de Trabajo

1. **Fase de Definición**:
   - El usuario ejecuta `logic-weaver init order-service --template workflow`
   - Crea `flows/order-service.yaml` con esqueleto inicial
   - La estructura YAML incluye: `nodes[]`, `edges[]`, `guards[]`, `actions[]`, `metadata`

2. **Fase de Diseño**:
   - Editar YAML con nodos (id, type: [state|action|decision|parallel], config)
   - Definir edges (source, target, condition expression)
   - Añadir guards (expresiones booleanas usando variables de contexto)
   - Definir actions (efectos secundarios, manejadores async)

3. **Fase de Validación**:
   - `logic-weaver validate` verifica:
     - Sin nodos huérfanos
     - Todas las variables referenciadas existen en el esquema de contexto
     - Sin dependencias circulares en expresiones de guard
     - Todas las actions tienen manejo de errores definido
     - Estados terminales marcados correctamente

4. **Fase de Generación**:
   - `logic-weaver generate` produce máquina de estado TypeScript XState
   - Incluye interfaz de contexto, tipos de evento, funciones de guard
   - Genera esqueleto de pruebas unitarias con todas las transiciones de estado

5. **Fase de Pruebas**:
   - Escribir archivos JSON de escenario: `{ \"initialContext\": {...}, \"events\": [...], \"expectedFinalState\": \"...\" }`
   - Ejecutar `logic-weaver test` - simula ruta de ejecución completa
   - Reporta ramas no probadas y nodos inalcanzables

6. **Fase de Despliegue**:
   - Desplegar en servicio XState o incrustar en aplicación
   - Registrar flujo con servicio runtime
   - Verificar health endpoint: `GET /api/flows/<name>/health`

7. **Fase de Monitoreo**:
   - Usar `logic-weaver logs --flow order-service --tail` para stream de ejecución
   - `logic-weaver stats` muestra conteos de ejecución, duración promedio, tasas de fallo

## Reglas de Oro

1. **Aislamiento de Contexto**: Cada flujo debe definir `contextSchema` con tipos estrictos. Nunca mutar estado externo directamente - usar actions que emitan eventos.
2. **Idempotencia**: Las actions deben ser idempotentes o incluir lógica de compensación. Las acciones de pago requieren manejadores de rollback.
3. **Pureza de Guard**: Las expresiones de guard deben ser funciones puras (sin efectos secundarios). Usar solo consultas de contexto.
4. **Garantía de Terminación**: Todo flujo debe tener al menos un estado terminal (éxito/fallo). Sin bucles infinitos.
5. **Límites de Error**: Definir transiciones `onError` para cada nodo. Los manejadores de error global deben existir en la raíz del flujo.
6. **Pinning de Versión**: Todos los flujos desplegados en producción deben estar versionados. Usar SHA de git en metadatos de despliegue.
7. **Cobertura de Pruebas**: Cada nodo y edge debe estar cubierto por al menos un escenario de prueba. `validate --coverage` lo exige.
8. **Documentación**: Cada nodo requiere campo `description`. Las condiciones de guard complejas requieren comentarios en línea en YAML.

## Ejemplos

### Ejemplo 1: Flujo de Procesamiento de Pedidos

**El usuario ejecuta:**
```bash
logic-weaver init order-processing --template workflow
```

**flows/order-processing.yaml:**
```yaml
name: order-processing
version: "1.0.0"
contextSchema:
  type: object
  properties:
    orderId: { type: string }
    userId: { type: string }
    paymentStatus: { type: string, enum: [pending, paid, failed] }
    inventoryReserved: { type: boolean }
    shippingAddress: { type: string }

nodes:
  - id: validateCart
    type: action
    description: Validar items de carrito y calcular total
    handler: src/actions/validateCart.ts
    timeout: 5000

  - id: paymentDecision
    type: decision
    description: Enrutar basado en método de pago
    expression: "context.paymentMethod === 'credit_card'"
    branches:
      - condition: true → target: processCreditCard
      - condition: false → target: processPayPal

edges:
  - source: validateCart
    target: paymentDecision
    condition: "context.cartValid === true"

  - source: processCreditCard
    target: capturePayment
    guard: "context.cardValidated === true"
```

**El usuario ejecuta:**
```bash
logic-weaver validate --file flows/order-processing.yaml
logic-weaver generate --lang ts --output src/flows/generated/
logic-weaver test --scenario tests/order-credit-success.json
```

**Salida (éxito):**
```
✓ Flow validation passed (12 checks)
✓ Generated: src/flows/generated/order-processing.ts
✓ Test scenario passed: order-credit-success (47 steps, 1.2s)
  States visited: validateCart → paymentDecision → processCreditCard → capturePayment → complete
```

### Ejemplo 2: Debug de Flujo Bloqueado

**El usuario ejecuta:**
```bash
logic-weaver test --file flows/refund-flow.yaml --scenario tests/refund-timeout.json
```

**Salida (fallo):**
```
✗ Test failed after 30s timeout
  Current state: waitingForBank
  Last action: initiateRefund (started 25s ago)
  Context: { refundId: "rf_123", amount: 99.99, status: "pending" }
  ⚠ Unhandled error: BankAPI timeout (code: GATEWAY_TIMEOUT)

Next steps:
1. Check refund handler logs: logic-weaver logs --flow refund-flow --action initiateRefund
2. Verify bank API connectivity: curl https://api.bank.com/health
3. Consider increasing timeout or adding retry logic to initiateRefund action
```

### Ejemplo 3: Visualización de Flujo

```bash
logic-weaver visualize --file flows/user-onboarding.yaml --format png --output docs/onboarding.png
# Genera diagrama 1920x1080 con colores de estado (verde=éxito, amarillo=pendiente, rojo=error)
```

### Ejemplo 4: Despliegue con Rollback

```bash
# Desplegar nueva versión
logic-weaver deploy --file flows/v2.yaml --runtime xstate --version v2.1.0

# Verificar despliegue
curl http://localhost:3000/api/flows/order-processing/health
# Response: {"status":"healthy","version":"v2.1.0","states":24}

# Rollback si se detectan problemas
logic-weaver rollback --flow order-processing --version v2.0.3
# Restaura versión anterior, actualiza endpoint health, activa invalidación de caché
```

## Comandos de Rollback

### Rollback Inmediato de Flujo
```bash
# Revertir a versión específica
logic-weaver rollback --flow <flow-name> --version <semantic-version|git-sha>

# Listar versiones disponibles
logic-weaver versions --flow <flow-name> --environment production

# Rollback vía override de configuración (emergencia)
echo '{"version":"1.0.0"}' > .logic-weaverrc.json
logic-weaver deploy --file flows/flow.yaml --force  # Usa versión sobrescrita
```

### Rollback de Base de Datos/Runtime
```bash
# Restaurar estado de ejecución de flujo desde backup
logic-weaver state restore --flow order-processing --backup /backups/state-2026-03-04T10:30:00Z.json

# Limpiar todas las instancias de flujo en memoria (usar con cautela)
logic-weaver instances purge --flow order-processing --older-than 24h

# Replay de eventos desde checkpoint
logic-weaver replay --flow order-processing --from-checkpoint cp_123 --to-checkpoint cp_125
```

### Rollback Completo (código + config + datos)
```bash
# Rollback completo del sistema (detiene servicio, revierte código, restaura DB)
logic-weaver full-rollback \
  --target-tag v1.2.0 \
  --database-backup /backups/db-2026-03-04.sql \
  --preserve-logs
```

## Solución de Problemas

| Problema | Comando | Resolución |
|----------|---------|------------|
| Validation falla: "undefined variable" | `logic-weaver validate --verbose` | Verificar que contextSchema incluye todas las variables referenciadas en guards |
| Código generado tiene errores | `logic-weaver generate --strict` | Asegurar que archivos handler exportan firma correcta: (context, event) => Promise<context> |
| Test cuelga/timeout | `logic-weaver test --step-debug` | Modo interactivo - paso a paso por cada transición, inspeccionar contexto |
| Visualización con nodos faltantes | `logic-weaver validate --orphans` | Corregir nodos desconectados (sin incoming/outgoing edges) |
| Deploy rechazado (conflicto de versión) | `logic-weaver diff --file v1.yaml v2.yaml` | Revisar cambios breaking; usar flag `--breaking` para permitir |
| Alto uso de memoria en runtime | `logic-weaver stats --memory` | Identificar flujos con contexto grande; limitar tamaño de contexto o añadir acciones de cleanup |
| Evaluación de guard lenta | `logic-weaver profile --flow <name>` | Optimizar expresiones de guard - cachear búsquedas costosas en contexto |

## Variables de Entorno

- `LOGIC_WEAVER_DEBUG=1`: Habilitar logging de debug (trazas de ejecución paso a paso)
- `LOGIC_WEAVER_CONFIG_PATH=/custom/path/.logic-weaverrc.json`: Sobrescribir ubicación de config
- `LOGIC_WEAVER_MAX_CONTEXT_SIZE=1048576`: Tamaño máximo de payload de contexto (bytes, default 1MB)
- `LOGIC_WEAVER_TIMEOUT=30000`: Timeout de acción por defecto (ms)
- `LOGIC_WEAVER_DISABLE_TELEMETRY=1`: Deshabilitar reporte de uso

## Dependencias y Requisitos

**Dependencias de build:**
- Node.js 18+ (ES2022 features)
- TypeScript 5.0+ (para generación)
- XState 5.0+ (opción runtime)
- Commander 11+ (framework CLI)

**Dependencias de runtime (elegir una):**
- `@xstate/react` (integración React)
- `xstate` (vanilla JS/Node)
- Runtime personalizado implementando interfaz `IRuntimeAdapter`

**Herramientas de desarrollo:**
- `dotenv` (carga de config)
- `yaml` (parsing de definición de flujo)
- `ajv` (validación de JSON schema)
- `js-yaml` (generación YAML para docs)

## Pasos de Verificación

1. **Post-generación**: `logic-weaver validate --file flows/<name>.yaml && logic-weaver test --all`
2. **Pre-despliegue**: `logic-weaver lint --file flows/` (verifica formato YAML, convenciones de nombres)
3. **Post-despliegue**: `curl -f http://localhost:3000/api/flows/<name>/health` debe retornar 200
4. **Smoke test**: Ejecutar flujo con entrada mínima: `echo '{}' | logic-weaver execute --stdin --flow <name>`
5. **Prueba de integración**: Ejecutar suite completa de escenarios: `logic-weaver test --dir tests/flows/ --parallel`

## Patrones Avanzados

### Manejo de Estados Paralelos
```yaml
nodes:
  - id: parallelProcess
    type: parallel
    branches:
      - id: shipping
        nodes: [reserveInventory, calculateTax]
      - id: billing
        nodes: [processPayment, sendInvoice]
```

### Compensación (Patrón Saga)
```yaml
nodes:
  - id: bookHotel
    type: action
    compensation: cancelHotelReservation
    timeout: 10000
```

### Activadores Basados en Eventos
```yaml
triggers:
  - type: webhook
    endpoint: /api/webhooks/order-created
    event: ORDER_CREATED
    mapping: "payload.orderId → context.orderId"
```
```