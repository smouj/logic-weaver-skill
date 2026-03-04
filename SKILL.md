---
name: Logic Weaver
slug: logic-weaver
version: 1.0.0
description: Weaves intricate logic flows into applications using state machines, decision trees, and workflow patterns
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

## Purpose

Logic Weaver creates, manages, and deploys complex logic flows for applications using declarative YAML configurations. It handles state machines, decision trees, and multi-step workflows with built-in validation, testing, and visualization.

### REAL Use Cases:

1. **E-commerce order processing**: Weave logic for cart validation → payment → inventory → shipping → notifications
2. **User onboarding flows**: Multi-step signup with conditional branches based on user type and data
3. **Game AI decision trees**: Complex NPC behavior with state persistence and transitions
4. **CI/CD pipeline orchestration**: Define deployment workflows with rollback triggers
5. **Form wizards**: Multi-page forms with skip logic, validation gates, and dynamic branching

## Scope

### Commands:

```bash
# Create a new logic flow from template
logic-weaver init <flow-name> --template <state-machine|decision-tree|workflow>

# Define nodes and transitions in YAML
logic-weaver design --file flows/order-processing.yaml

# Validate logic flow integrity
logic-weaver validate --file flows/order-processing.yaml --strict

# Generate TypeScript/JavaScript implementation
logic-weaver generate --file flows/order-processing.yaml --lang ts --output src/flows/

# Visualize flow as SVG/PNG
logic-weaver visualize --file flows/order-processing.yaml --format svg --output docs/flow-diagram.svg

# Test flow with simulation inputs
logic-weaver test --file flows/order-processing.yaml --scenario tests/order-success.json

# Deploy flow to runtime (XState service, custom runtime)
logic-weaver deploy --file flows/order-processing.yaml --runtime xstate --endpoint /api/flows/order

# Compare two flow versions
logic-weaver diff --file v1.yaml v2.yaml --format table

# List all flows with status
logic-weaver list --environment production

# Execute single step manually (debug)
logic-weaver step --flow order-processing --state PaymentFailed --input '{"orderId":"123"}'
```

### File Patterns:

- Flow definitions: `**/flows/*.yaml` or `**/logic/*.yaml`
- Test scenarios: `**/tests/**/*.json` (paired with flow name)
- Generated code: `src/flows/generated/`
- Config: `.logic-weaverrc.json` or `logic-weaver.config.yaml`

## Work Process

1. **Definition Phase**:
   - User runs `logic-weaver init order-service --template workflow`
   - Creates `flows/order-service.yaml` with initial skeleton
   - YAML structure includes: `nodes[]`, `edges[]`, `guards[]`, `actions[]`, `metadata`

2. **Design Phase**:
   - Edit YAML with nodes (id, type: [state|action|decision|parallel], config)
   - Define edges (source, target, condition expression)
   - Add guards (boolean expressions using context variables)
   - Define actions (side-effects, async handlers)

3. **Validation Phase**:
   - `logic-weaver validate` checks:
     - No orphaned nodes
     - All referenced variables exist in context schema
     - No circular dependencies in guard expressions
     - All actions have proper error handling defined
     - Terminal states marked correctly

4. **Generation Phase**:
   - `logic-weaver generate` produces TypeScript XState machine
   - Includes context interface, event types, guard functions
   - Generates unit test skeleton with all state transitions

5. **Testing Phase**:
   - Write scenario JSON files: `{ "initialContext": {...}, "events": [...], "expectedFinalState": "..." }`
   - Run `logic-weaver test` - simulates full execution path
   - Reports untested branches and unreachable nodes

6. **Deployment Phase**:
   - Deploy to XState service or embed in application
   - Register flow with runtime service
   - Verify health endpoint: `GET /api/flows/<name>/health`

7. **Monitoring Phase**:
   - Use `logic-weaver logs --flow order-service --tail` to stream execution
   - `logic-weaver stats` shows execution counts, avg duration, failure rates

## Golden Rules

1. **Context Isolation**: Each flow must define `contextSchema` with strict types. Never mutate external state directly - use actions that emit events.
2. **Idempotency**: Actions must be idempotent or include compensation logic. Payment actions require rollback handlers.
3. **Guard Purity**: Guard expressions must be pure functions (no side effects). Use context lookups only.
4. **Termination Guarantee**: Every flow must have at least one terminal state (success/failure). No infinite loops.
5. **Error Boundaries**: Define `onError` transitions for every node. Global error handlers must exist at flow root.
6. **Version Pinning**: All flows deployed to production must be versioned. Use git SHA in deployment metadata.
7. **Testing Coverage**: Every node and edge must be covered by at least one test scenario. `validate --coverage` enforces this.
8. **Documentation**: Every node requires `description` field. Complex guard conditions require inline comments in YAML.

## Examples

### Example 1: Order Processing Flow

**User runs:**
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
    description: Validate cart items and calculate total
    handler: src/actions/validateCart.ts
    timeout: 5000

  - id: paymentDecision
    type: decision
    description: Route based on payment method
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

**User runs:**
```bash
logic-weaver validate --file flows/order-processing.yaml
logic-weaver generate --lang ts --output src/flows/generated/
logic-weaver test --scenario tests/order-credit-success.json
```

**Output (success):**
```
✓ Flow validation passed (12 checks)
✓ Generated: src/flows/generated/order-processing.ts
✓ Test scenario passed: order-credit-success (47 steps, 1.2s)
  States visited: validateCart → paymentDecision → processCreditCard → capturePayment → complete
```

### Example 2: Debugging a Lock Flow

**User runs:**
```bash
logic-weaver test --file flows/refund-flow.yaml --scenario tests/refund-timeout.json
```

**Output (failure):**
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

### Example 3: Visualizing Flow

```bash
logic-weaver visualize --file flows/user-onboarding.yaml --format png --output docs/onboarding.png
# Generates 1920x1080 diagram with state colors (green=success, yellow=pending, red=error)
```

### Example 4: Deployment with Rollback

```bash
# Deploy new version
logic-weaver deploy --file flows/v2.yaml --runtime xstate --version v2.1.0

# Verify deployment
curl http://localhost:3000/api/flows/order-processing/health
# Response: {"status":"healthy","version":"v2.1.0","states":24}

# Rollback if issues detected
logic-weaver rollback --flow order-processing --version v2.0.3
# Restores previous version, updates health endpoint, triggers cache invalidation
```

## Rollback Commands

### Immediate Flow Rollback
```bash
# Revert to specific version
logic-weaver rollback --flow <flow-name> --version <semantic-version|git-sha>

# List available versions
logic-weaver versions --flow <flow-name> --environment production

# Rollback via config override (emergency)
echo '{"version":"1.0.0"}' > .logic-weaverrc.json
logic-weaver deploy --file flows/flow.yaml --force  # Uses overridden version
```

### Database/Runtime Rollback
```bash
# Restore flow execution state from backup
logic-weaver state restore --flow order-processing --backup /backups/state-2026-03-04T10:30:00Z.json

# Clear all in-memory flow instances (use cautiously)
logic-weaver instances purge --flow order-processing --older-than 24h

# Replay events from checkpoint
logic-weaver replay --flow order-processing --from-checkpoint cp_123 --to-checkpoint cp_125
```

### Complete Rollback (code + config + data)
```bash
# Full system rollback (stops service, reverts code, restores DB)
logic-weaver full-rollback \
  --target-tag v1.2.0 \
  --database-backup /backups/db-2026-03-04.sql \
  --preserve-logs
```

## Troubleshooting

| Issue | Command | Resolution |
|-------|---------|------------|
| Validation fails: "undefined variable" | `logic-weaver validate --verbose` | Check contextSchema includes all variables referenced in guards |
| Generated code has errors | `logic-weaver generate --strict` | Ensure handler files export correct signature: (context, event) => Promise<context> |
| Test hangs/timeout | `logic-weaver test --step-debug` | Interactive mode - step through each transition, inspect context |
| Visualization missing nodes | `logic-weaver validate --orphans` | Fix disconnected nodes (no incoming/outgoing edges) |
| Deploy rejected (version conflict) | `logic-weaver diff --file v1.yaml v2.yaml` | Review breaking changes; use `--breaking` flag to allow |
| High memory usage in runtime | `logic-weaver stats --memory` | Identify flows with large context; limit context size or add cleanup actions |
| Guard evaluation slow | `logic-weaver profile --flow <name>` | Optimize guard expressions - cache expensive lookups in context |

## Environment Variables

- `LOGIC_WEAVER_DEBUG=1`: Enable debug logging (step-by-step execution traces)
- `LOGIC_WEAVER_CONFIG_PATH=/custom/path/.logic-weaverrc.json`: Override config location
- `LOGIC_WEAVER_MAX_CONTEXT_SIZE=1048576`: Maximum context payload size (bytes, default 1MB)
- `LOGIC_WEAVER_TIMEOUT=30000`: Default action timeout (ms)
- `LOGIC_WEAVER_DISABLE_TELEMETRY=1`: Disable usage reporting

## Dependencies & Requirements

**Build dependencies:**
- Node.js 18+ (ES2022 features)
- TypeScript 5.0+ (for generation)
- XState 5.0+ (runtime option)
- Commander 11+ (CLI framework)

**Runtime dependencies (choose one):**
- `@xstate/react` (React integration)
- `xstate` (vanilla JS/Node)
- Custom runtime implementing `IRuntimeAdapter` interface

**Development tools:**
- `dotenv` (config loading)
- `yaml` (flow definition parsing)
- `ajv` (JSON schema validation)
- `js-yaml` (YAML generation for docs)

## Verification Steps

1. **Post-generation**: `logic-weaver validate --file flows/<name>.yaml && logic-weaver test --all`
2. **Pre-deployment**: `logic-weaver lint --file flows/` (checks YAML formatting, naming conventions)
3. **Post-deployment**: `curl -f http://localhost:3000/api/flows/<name>/health` must return 200
4. **Smoke test**: Execute flow with minimal input: `echo '{}' | logic-weaver execute --stdin --flow <name>`
5. **Integration test**: Run full scenario suite: `logic-weaver test --dir tests/flows/ --parallel`

## Advanced Patterns

### Parallel State Handling
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

### Compensation (Saga Pattern)
```yaml
nodes:
  - id: bookHotel
    type: action
    compensation: cancelHotelReservation
    timeout: 10000
```

### Event-Driven Triggers
```yaml
triggers:
  - type: webhook
    endpoint: /api/webhooks/order-created
    event: ORDER_CREATED
    mapping: "payload.orderId → context.orderId"
```
```