<div align="center">

<img src="https://raw.githubusercontent.com/Run-IQ/.github/main/profile/logo.png" width="120" alt="Run-IQ" />

# Run-IQ

**Parametric Rule Engine for regulated domains**

*Deterministic · Auditable · Extensible*

[![MIT License](https://img.shields.io/badge/core-MIT-00D4FF?style=flat-square)](https://github.com/Run-IQ/core/blob/master/LICENSE)
[![MIT License](https://img.shields.io/badge/plugin--sdk-MIT-00D4FF?style=flat-square)](https://github.com/Run-IQ/plugin-sdk/blob/master/LICENSE)
[![Source Available](https://img.shields.io/badge/plugins-Source--Available-f59e0b?style=flat-square)](https://run-iq.org/pricing)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6?style=flat-square)](https://www.typescriptlang.org/)
[![Website](https://img.shields.io/badge/run--iq.org-→-0d1528?style=flat-square&logo=globe)](https://run-iq.org)

</div>

---

## What is Run-IQ?

Run-IQ implements the **PPE specification** — a Parametric Policy Engine that applies versioned rules to input data and produces a traceable, deterministic, legally auditable result.

It is not a tax engine. Not an HR engine. It is a **generic rule engine** on which domain plugins are registered. The same engine powers fiscal calculations, payroll, banking fees, insurance premiums, and government subsidies.

```
Same input + Same rules + Same engine version = Same result. Always.
```

---

## The Problem

Regulated domains — tax, payroll, banking, insurance — share a universal challenge:

- Rules change frequently (new law, new rate, new threshold)
- Every calculation must be reproducible years later for audit
- A small bug in a calculation has legal consequences
- The same logic applies across multiple countries with local variations

Most teams hardcode these rules into application logic. When the law changes, they deploy. When audited, they hope the old version still runs. When scaling to a new country, they copy-paste and diverge.

Run-IQ solves this at the infrastructure level.

---

## How It Works

```
┌─────────────────────────────────────────┐
│            Your Application             │
│   (rules in DB · snapshot adapter)      │
└────────────────┬────────────────────────┘
                 │ rules + input
                 ▼
┌─────────────────────────────────────────┐
│           @run-iq/core  (MIT)           │
│                                         │
│  validate → filter → resolve → execute  │
│        → trace → snapshot               │
└──────┬─────────────────────┬────────────┘
       │                     │
       ▼                     ▼
┌─────────────┐     ┌─────────────────────┐
│  @run-iq/   │     │  @run-iq/plugin-sdk │
│  dsl-*      │     │  BasePlugin         │
│  (MIT)      │     │  BaseModel          │
│             │     │  PluginTester       │
│  jsonlogic  │     └──────────┬──────────┘
│  cel        │                │
└─────────────┘                ▼
                    ┌──────────────────────┐
                    │  @run-iq/plugin-*    │
                    │  (Source-Available)  │
                    │                      │
                    │  plugin-fiscal       │
                    │  plugin-payroll      │
                    └──────────────────────┘
```

The engine **knows no domain**. It receives rules, evaluates conditions, dispatches to registered calculation models, and archives a cryptographically signed snapshot. Domain knowledge lives entirely in plugins.

---

## Core Guarantees

| Guarantee | Mechanism |
|---|---|
| **Determinism** | Same input × same rules × same version = same result |
| **Immutability** | Rules are checksummed. Any mutation is detected and rejected |
| **Idempotence** | `requestId` prevents any calculation from running twice |
| **Audit trail** | Every calculation produces a full snapshot archived by the host |
| **Legal reproducibility** | Snapshots store complete rule data — not IDs — for replay in 5 years |
| **Integrity** | SHA-256 checksum on every snapshot detects post-archival tampering |

---

## Packages

### Open Source (MIT)

| Package | Description |
|---|---|
| [`@run-iq/core`](https://github.com/Run-IQ/core) | The engine — pipeline, registries, snapshot, security |
| [`@run-iq/plugin-sdk`](https://github.com/Run-IQ/plugin-sdk) | Kit for building plugins — `BasePlugin`, `BaseModel`, `PluginTester` |
| [`@run-iq/dsl-jsonlogic`](https://github.com/Run-IQ/dsl-jsonlogic) | JSONLogic condition evaluator — rules stored as JSON |
| [`@run-iq/dsl-cel`](https://github.com/Run-IQ/dsl-cel) | CEL (Common Expression Language) evaluator |
| [`@run-iq/server`](https://github.com/Run-IQ/server) | HTTP REST wrapper — polyglot integration |
| [`@run-iq/cli`](https://github.com/Run-IQ/cli) | Command-line tool — `run-iq evaluate --rules r.json --input i.json` |

### Source-Available

| Package | Description |
|---|---|
| [`@run-iq/plugin-fiscal`](https://github.com/Run-IQ/plugin-fiscal) | Fiscal domain — TVA, IRPP, IS, multi-jurisdiction, multi-country |
| [`@run-iq/plugin-payroll`](https://github.com/Run-IQ/plugin-payroll) | Payroll domain — gross salary, social contributions, commissions, bonuses |

> Source-Available packages are free to read and evaluate. Commercial use requires a license. See [run-iq.org/pricing](https://run-iq.org/pricing).

---

## Quick Start

```bash
npm install @run-iq/core @run-iq/dsl-jsonlogic @run-iq/plugin-sdk
```

```typescript
import { PPEEngine } from "@run-iq/core"
import { JsonLogicEvaluator } from "@run-iq/dsl-jsonlogic"

// Implement ISnapshotAdapter for your database
class MySnapshotAdapter implements ISnapshotAdapter {
  async save(snapshot) { /* store in your DB */ return snapshot.id }
  async get(id)        { /* retrieve from your DB */ }
  async exists(reqId)  { /* check if already processed */ }
}

// Instantiate the engine
const engine = new PPEEngine({
  plugins  : [],                          // add domain plugins here
  dsls     : [new JsonLogicEvaluator()],  // add DSL evaluators here
  snapshot : new MySnapshotAdapter(),
  strict   : true
})

// Evaluate rules against input
const result = await engine.evaluate(rules, {
  requestId : crypto.randomUUID(),
  data      : { amount: 1500000 },
  meta      : { tenantId: "org-123" }
})

console.log(result.value)       // calculated result
console.log(result.snapshotId)  // archived for audit
console.log(result.trace)       // full step-by-step trace
```

---

## Building a Plugin

```bash
npm install @run-iq/core @run-iq/plugin-sdk
```

```typescript
import { BasePlugin, BaseModel, SchemaValidator, PluginTester } from "@run-iq/plugin-sdk"

class FlatRateModel extends BaseModel {
  readonly name = "FLAT_RATE"
  readonly version = "1.0.0"

  validateParams(params) {
    return SchemaValidator.validate(params, {
      rate : { type: "number", min: 0, max: 1 },
      base : { type: "string" }
    })
  }

  calculate(input, rule, params) {
    return (input[params.base] as number) * params.rate
    // Pure function — no side effects, no state
  }
}

class MyPlugin extends BasePlugin {
  readonly name    = "@my-org/my-plugin"
  readonly version = "1.0.0"
  models = [new FlatRateModel()]
  // BasePlugin auto-registers all models on init
}

// Validate conformance to PPE spec
const tester = new PluginTester(new MyPlugin())
const report = await tester.runAll(sampleInput, sampleRules)
// → assertDeterminism · assertImmutability · assertNoSideEffects
```

---

## Use Cases

Run-IQ is domain-agnostic. The same engine powers:

- **Tax** — VAT, income tax, corporate tax, customs duties, multi-country
- **Payroll** — gross-to-net, social contributions, commissions, performance bonuses
- **Banking** — dynamic fee schedules, regulated interest rates
- **Insurance** — premium calculation, risk scoring
- **Government** — subsidy eligibility, social benefit calculation
- **Energy** — tiered billing, consumption thresholds
- **Commerce** — dynamic pricing, volume discounts, royalties

---

## Design Principles

**The engine knows no domain.**
`rule.model` is an opaque string resolved through the `ModelRegistry`. `rule.params` is `unknown` passed as-is to the model. Nothing fiscal, payroll-specific, or domain-related ever enters the core.

**The snapshot is non-negotiable.**
In `strict` mode, the engine will not return a result if the snapshot fails to save. A calculation without an archived trace has no legal value.

**Immutability is architectural.**
Plugin hooks receive `ReadonlyArray<Rule>`. Any attempted mutation is detected in development by `BasePlugin`'s deep-freeze guards. Determinism is enforced structurally, not by convention.

**The host owns its data.**
The engine stores nothing. It consumes injected rules, computes a result, and calls your `ISnapshotAdapter`. Storage, persistence, and access control are entirely yours.

---

## Ecosystem

```
Run-IQ/core            ← The engine
Run-IQ/plugin-sdk      ← Build your own plugin
Run-IQ/dsl-jsonlogic   ← JSONLogic conditions
Run-IQ/dsl-cel         ← CEL conditions
Run-IQ/plugin-fiscal   ← Fiscal domain (West Africa · expanding)
Run-IQ/plugin-payroll  ← Payroll domain
Run-IQ/server          ← HTTP API wrapper
Run-IQ/cli             ← Local rule testing
```

---

## Roadmap

- [x] Core engine — pipeline, snapshot, DSL registry
- [x] JSONLogic DSL evaluator
- [x] Plugin SDK — BasePlugin, BaseModel, PluginTester
- [x] `@run-iq/plugin-fiscal` — fiscal domain, West Africa
- [ ] `@run-iq/plugin-payroll` — payroll domain (in progress)
- [ ] `@run-iq/server` — HTTP REST wrapper
- [ ] `@run-iq/cli` — command-line tool
- [ ] `@run-iq/dsl-cel` — CEL evaluator
- [ ] `@run-iq/simulation` — hypothetical rule simulation engine
- [ ]  `@run-iq/mcp-server`
- [ ] Community plugin registry at run-iq.org

---

## License

| Package | License |
|---|---|
| `@run-iq/core` | [MIT](https://github.com/Run-IQ/core/blob/main/LICENSE) |
| `@run-iq/plugin-sdk` | [MIT](https://github.com/Run-IQ/plugin-sdk/blob/main/LICENSE) |
| `@run-iq/dsl-*` | [MIT](https://github.com/Run-IQ/dsl-jsonlogic/blob/main/LICENSE) |
| `@run-iq/server` | [MIT](https://github.com/Run-IQ/server/blob/main/LICENSE) |
| `@run-iq/cli` | [MIT](https://github.com/Run-IQ/cli/blob/main/LICENSE) |
| `@run-iq/plugin-*` | [Source-Available](https://run-iq.org/license) — commercial use requires license |

---

<div align="center">

**[run-iq.org](https://run-iq.org)** · **[Documentation](https://run-iq.org/docs)** · **[Pricing](https://run-iq.org/pricing)**

*Built with precision. Designed for regulated domains.*

</div>
