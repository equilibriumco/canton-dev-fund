## Development Fund Proposal

**Author:** [FILL IN]
**Status:** Draft
**Created:** 2026-03-06

---

## Abstract

Daml Forge is an open-source CLI toolkit for Canton Network: three small Rust binaries that cover the gap between `dpm build` and a running application. `dforge node` starts a local Canton environment in one command, with pre-funded parties and a persistent alias book. `dforge deploy` uploads a `.dar`, validates its dependencies, and registers a human-readable name for the package. `dforge call` exercises choices and queries state using those aliases, replacing the multi-step JSON-API and gRPC workflows that contract interaction currently requires.

The toolkit is shaped after Foundry. A fast native CLI orchestrates `damlc` (via `dpm`) and the Canton sandbox rather than reimplementing them, which keeps Daml Forge complementary to the existing Daml SDK rather than competing with it. `dpm build` and `dpm test` remain the backbone of the build and test cycle; Daml Forge adds the layer above them that the ecosystem currently has to assemble by hand.

The goal is a tight inner loop. From a fresh directory to a running contract interaction in three commands, with no JVM startup in the path, no codegen step, and no 68-character party IDs to copy and paste.

---

## Specification

### 1. Objective

Daml Forge gives Canton developers a single tool for the everyday cases: spin up a local environment, deploy a package, exercise a choice, query state. Each of those is currently stitched together from JSON-API calls, Docker compositions, and console sessions. The toolkit collapses each into one command and threads a single alias system through all of them, so a party named once stays a party named everywhere.

The opinion the toolkit takes is that the interaction surface is the product. `dforge` makes deliberate choices about what to surface (short, human-readable aliases instead of 68- and 136-character hex IDs), what to manage automatically (party allocation, OAuth tokens, deploy idempotency, choice and template inference from `.dalf` artifacts), and what to delegate (compilation to `dpm`, the runtime to Canton itself). The result is the shape of Foundry's `cast`, `forge`, and `anvil` for Ethereum, sized for the ergonomics that incoming developers expect when they walk into their first Daml repository.

The tooling is built to be automatable from day one. Every command can emit structured JSON, expose its argument schema, and run in milliseconds, which matters as much for CI scripts and AI coding agents as it does for an interactive terminal. The same alias-resolution and short-ID conventions that make the CLI pleasant in a shell make it tractable in agentic workflows without additional glue.

**Intended Outcome:** A first contract interaction within three commands of a fresh directory, on standard developer hardware.

### 2. Implementation Mechanics

Daml Forge is three CLI tools written in Rust that talk to Canton over the standard public APIs (JSON API v2 and the gRPC Ledger API). The Rust binaries handle everything user-facing: argument parsing, alias and short-ID resolution, output formatting, native `.dar` and `.dalf` parsing, and the address-book and package-index files that thread through every command. They orchestrate the existing compiler and node rather than reimplementing them. `dpm build` (and through it `damlc`) compiles `.dar` files; Canton itself runs the local sandbox that `dforge node` manages as a child process. Workflows that target an already-running participant — `dforge deploy`, `dforge call`, `dforge query` — need no JVM at all on the developer's machine. The toolkit ships as a single static binary per platform, with no runtime dependencies.

#### Alias & Resolution System

The core differentiator. When a developer runs `dforge call @ore-token GetView --as alice`:

1. Resolve `alice` → `alice::1220abc...` (party alias from address book)
2. Resolve `@ore-token` → `00abc123...` (contract alias or prefix match)
3. Discover `GetView` is on `Asset` interface, resolve interface package ID
4. Execute via JSON API v2 or gRPC, return labeled output

Aliases are scoped per network. Auto-aliasing occurs on `dforge deploy` (packages), `dforge node` startup / party allocation (parties), and `dforge call --create` (contracts). Prefix-based resolution allows shorthand whenever the result is unambiguous, similar to git's short SHA-1 hashes.

Choice resolution inspects `.dalf` artifacts from uploaded packages. Template choices are prioritized; interface choices are resolved when unambiguous. Ambiguous references produce helpful error messages listing all possibilities.

#### Relationship to Existing Tools

`dpm build` and `dpm test` remain the backbone of the compile-and-test cycle. Daml Forge delegates to them rather than duplicating their work: `dforge deploy --build` invokes `dpm build`, and `dpm inspect-dar` feeds the package metadata index. The test loop is left untouched. All ledger interaction uses Canton's public JSON API v2 (CIP-0062) and the gRPC Ledger API — no internal endpoints. Canton Console (the Scala REPL) and Daml Forge target different audiences and do not overlap; a developer who prefers REPL-driven exploration can keep using it.

#### `dforge node` — Local Canton Environment

A single command that starts a ready-to-use Canton development environment:

```bash
$ dforge node
Starting Canton sandbox on port 7575...
  Participant: localhost:7575 (JSON API v2)
  Sequencer:   localhost:7576
  Mediator:    localhost:7577

Pre-funded parties:
  alice   alice-a1b2::1220... (alias: alice)
  bob     bob-c3d4::1220...   (alias: bob)
  bank    bank-e5f6::1220...  (alias: bank)

Address book saved to .dforge/parties.json
Ready in 4.2s.
```

Key capabilities:
- **Single-process mode**: Participant + Sequencer + Mediator in one process (no Docker required for basic development)
- **Pre-funded parties**: Configurable set of parties automatically registered, authorized, and funded
- **Address book**: Persistent alias mapping (`alice` → full party ID) used across all CLI commands
- **Runtime package loading**: Upload `.dar` files to a running node without restart
- **Configurable topologies**: Single-party (default), multi-party, multi-domain via `dforge node --topology multi-domain`
- **OAuth2 support**: `dforge node --with-oauth` launches full OAuth2-enabled environment with auto-managed credentials
- **Splice stack**: `dforge node --with-splice` deploys Super Validator, Scan, and Wallet for Amulet/CIP-56 token development
- **Deterministic mode**: Reproducible state for CI/CD (`dforge node --deterministic`)
- **Fast reset**: `dforge node --reset` without full re-initialization

#### `dforge call` — Contract Interaction CLI

Single commands for ledger interaction, analogous to Foundry's `cast`:

```bash
# Create a contract
$ dforge call --create OreToken @bank @alice 100.00 --as bank,alice
✓ Contract created: OreToken
  Contract ID: 00abc123... (alias: @ore-token)
  Transaction ID: tx-123456

# Exercise a choice on a contract
$ dforge call @ore-token Split 30.0 --as alice
✓ Choice exercised: Split
  Result: (newCid1, newCid2)

# Exercise a non-consuming choice, output as JSON
$ dforge call @ore-token GetView --as alice --json
{
  "assetOwner": "alice::1220abc...",
  "description": "Magic Ore",
  "quantity": 70.0
}

# Query active contracts
$ dforge query contracts --template OreToken --as alice
  #1  Main:OreToken  owner=alice  grams=100.0
  #2  Main:OreToken  owner=alice  grams=50.0

# List uploaded packages with human-readable names
$ dforge query packages
  #1  ore-bank-main-1.0.0      f6f30cd7...
  #2  ore-bank-interfaces-1.0  e503a7f4...

# Multi-party authorization
$ dforge call @proposal Accept --as alice,bob --read-as auditor
```

Key capabilities:
- **Short IDs**: Reference contracts by alias (`@ore-token`), parties by name (`alice`), packages by name — no 68-136 character hex strings
- **Formatted output**: Human-readable by default; `--json` for scripting and AI agent consumption
- **Field masking**: `--fields contractId,payload.grams` to limit output size
- **Dry-run mode**: `--dry-run` to simulate mutations before committing
- **Queryable schemas**: `dforge schema <command>` exposing parameters and types as JSON for agentic workflows
- **Verbose tracing**: `--verbose` shows resolution steps, auth state, and API calls for debugging

#### `dforge deploy` — Package Deployment

```bash
# Deploy to local node with pre-validation
$ dforge deploy .daml/dist/my-app-1.0.0.dar
  Validating package...
  Checking dependencies...
  Uploading to localhost:7575...
  Done. Package ID: a1b2c3d4... (alias: @my-app)

# Build via dpm then deploy in one command
$ dforge deploy --build ./main/

# Deploy to a named environment with traffic cost estimation
$ dforge deploy .daml/dist/my-app-1.0.0.dar --env testnet
  Target: testnet (participant.example.com:7575)
  Estimated traffic cost: 0.42 CC (CIP-0089)
  Proceed? [y/N]
```

Key capabilities:
- Pre-deployment validation: check package compatibility, verify dependencies are uploaded
- `--build` delegation to `dpm build` before upload
- Environment management: named configurations in `dforge.yaml` for local/testnet/mainnet
- Traffic cost estimation using Canton 3.4 APIs (CIP-0089)
- Idempotent deploys: skip already-uploaded packages

#### Token & Wallet Support

CIP-56 compliant token operations integrated with the Splice stack, making Amulet and custom token management accessible from the command line:

```bash
# Amulet/CC balance
$ dforge wallet balance --as alice
ASSET          BALANCE      LOCKED    AVAILABLE
Canton Coin    1,250.50 CC  100.00    1,150.50

# Transfer
$ dforge wallet transfer --to bob --amount 50 --as alice

# CIP-56 token holdings
$ dforge wallet holdings --token "Gold Token" --as alice

# UTXO consolidation
$ dforge wallet merge --token "Canton Coin" --as alice
```

These commands wrap the Splice stack's Scan and Registry APIs, increasing the accessibility and adoption of Canton's token infrastructure.

#### AI Agent Support

Designed for agentic workflows from the start:
- Structured `--json` output on all commands for machine-readable responses
- `--fields` for field masking to protect LLM context windows
- `--dry-run` for safe exploration before mutations
- `dforge schema <command>` exposing parameters, types, and constraints as queryable JSON
- Input validation with clear error messages for malformed parameters
- Pagination via NDJSON for streaming large result sets

#### From Zero to Interaction in Three Commands

A read-only choice exercise on a fresh node currently takes around five API calls and fifty lines of curl across the JSON API. With Daml Forge:

```bash
$ dforge node --party alice --party bank
$ dforge deploy --build
$ dforge call @ore-token GetView --as alice --json
```

Auth, party resolution, package discovery, and ID management are all handled inside the three commands.

### 3. Architectural Alignment

**Canton Protocol Compatibility:**
- All interaction goes through Canton's official JSON API v2 (CIP-0062) and gRPC Ledger API — no protocol modifications or custom endpoints
- Supports Smart Contract Upgrades (SCU) introduced in Canton 3.3 (CIP-0062)
- Traffic cost estimation leverages Canton 3.4 APIs (CIP-0089)
- Local node bootstrapping follows Canton's standard participant/sequencer/mediator architecture

**CIP Standards Support:**
- CIP-0056 (Token Standard): wallet commands implement token balance, transfer (FOP), allocation (DVP), and UTXO management
- CIP-0089: Traffic cost estimation integrated into `dforge deploy`
- CIP-0062: JSON API v2 as the primary interaction protocol

**Ecosystem Integration:**
- Complements the existing Daml SDK — delegates compilation and testing to `dpm`
- Output formats compatible with existing Canton Console and daml-shell workflows
- Open-source under Apache-2.0, hosted in Canton Foundation GitHub organization

The technology choice (Rust on the CLI side, JVM on the sandbox side) is covered in detail under [Why Rust?](#why-rust).

### 4. Backward Compatibility

*No backward compatibility impact.* Daml Forge is a new developer tool that operates alongside existing Daml SDK and Canton tooling. It produces standard `.dar` files and communicates exclusively through Canton's public APIs. No changes to Canton protocol, Daml language, or existing tooling are required.

### 5. Security Considerations

- All ledger interaction uses Canton's standard authenticated APIs (JSON API v2, gRPC Ledger API) — no internal or private API access
- OAuth2/JWT tokens are stored in user-local configuration directories with restricted file permissions (`~/.dforge/credentials/`)
- No credential storage in plaintext; tokens are acquired via standard OAuth2 flows and cached with expiry tracking
- `dforge node` runs the Canton sandbox as a local process with no network exposure beyond localhost by default
- The CLI validates all user input before constructing API requests, preventing malformed payloads from reaching the node
- Open-source codebase enables community security review

---

## Milestones and Deliverables

### Milestone 1: Core CLI & Local Node (`dforge node`)
- **Estimated Delivery:** T+10 weeks
- **Focus:** Local Canton environment, address book, project configuration
- **Deliverables / Value Metrics:**
  - `dforge node` — single-command Canton sandbox with pre-funded parties, address book, configurable topologies, OAuth2 support
  - `dforge node --reset` — fast state reset without full restart
  - `dforge node --with-splice` — Splice stack deployment (SV, Scan, Wallet) for token development
  - Runtime `.dar` upload to running node
  - `dforge.yaml` project configuration format
  - Party management: `dforge party new`, `dforge party ensure` (idempotent get-or-create)
  - Auth management: `dforge auth login`, `dforge auth token`, `dforge auth status`
  - Developer documentation and getting-started guide
  - **Success metric:** Time from install to first running contract < 5 minutes

### Milestone 2: Contract Interaction CLI (`dforge call`)
- **Estimated Delivery:** T+20 weeks
- **Focus:** Ledger interaction, alias resolution, package discovery
- **Deliverables / Value Metrics:**
  - `dforge call` — exercise choices and create contracts with alias/short-ID support
  - `dforge query` — query contracts, parties, packages, transactions with filtering
  - **Alias & Resolution System:** persistent aliases for contracts (`@ore-token`), parties (`alice`), packages (`@ore-bank-main`); auto-aliasing on deploy/create; prefix resolution; interface-to-package discovery via `.dalf` inspection; meaningful ambiguity errors
  - `dforge deploy` — package upload with pre-validation, dependency checking, `--build` delegation to `dpm`
  - `dforge config` — default networks, participants, parties, output format
  - `--json` output mode, `--fields` masking, `--dry-run` simulation, `--verbose` tracing
  - `dforge schema <command>` — queryable JSON schemas for AI agent integration
  - Multi-party authorization via `--as` and `--read-as` flags
  - **Success metric:** Exercise a contract choice in 1 command (vs. current 3+ API calls)

### Milestone 3: Token/Wallet Support, Deployment Tooling & Polish
- **Estimated Delivery:** T+28 weeks
- **Focus:** CIP-56 token operations, environment management, CI/CD integration, documentation
- **Deliverables / Value Metrics:**
  - `dforge wallet balance` and `dforge wallet holdings` for Amulet (CC) and CIP-56 tokens
  - `dforge wallet transfer` for FOP transfers, `dforge wallet merge` for UTXO consolidation
  - Integration with Scan/Registry APIs for token metadata and transfer instructions
  - `dforge deploy --env <name>` — named environment management (local/testnet/mainnet) in `dforge.yaml`
  - Traffic cost estimation (CIP-0089) in deployment workflow
  - CI/CD templates (GitHub Actions, GitLab CI) for Daml Forge workflows
  - Complete user documentation, tutorials, and demo project
  - **Success metric:** Full token transfer workflow (balance → transfer → verify) in 2 commands

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone
- Demonstrated functionality or operational readiness
- Documentation and knowledge transfer provided
- Alignment with stated value metrics

**Project-specific acceptance conditions:**

- All CLI tools pass end-to-end integration tests against Canton sandbox and localnet
- `dforge node` starts a usable environment in under 30 seconds on standard developer hardware
- `dforge call` completes a contract choice exercise in a single command with alias/short-ID support
- All tools communicate exclusively through Canton's public JSON API v2 and gRPC APIs (no internal/private APIs)
- Source code published under Apache-2.0 in Canton Foundation GitHub organization
- Getting-started guide enables a new developer to go from install to first contract interaction within 5 minutes
- CI/CD templates demonstrated working in at least one public pipeline
- `dforge wallet` operations execute against both Amulet (CC) and custom CIP-56 tokens on a Splice-enabled localnet

---

## Funding

**Total Funding Request:** [FILL IN] CC

Estimated team: [FILL IN] engineers over 28 weeks (~7 months).

### Payment Breakdown by Milestone
- Milestone 1 (Core CLI & Local Node): [FILL IN] CC upon committee acceptance (~35% of total)
- Milestone 2 (Contract Interaction CLI): [FILL IN] CC upon committee acceptance (~40% of total)
- Milestone 3 (Token/Wallet & Polish): [FILL IN] CC upon final release and acceptance (~25% of total)

### Volatility Stipulation
The project duration is **greater than 6 months** (estimated 28 weeks):
The grant is denominated in fixed Canton Coin and will require a re-evaluation at the 6-month mark. Should the project timeline extend beyond the estimated duration due to Committee-requested scope changes, any remaining milestones must be renegotiated to account for significant USD/CC price volatility.

---

## Co-Marketing

Upon release of each milestone, the implementing entity will collaborate with Canton Foundation on:

- Announcement coordination for each milestone release
- Technical blog post series: "From Foundry to Daml Forge" — a migration guide for EVM developers transitioning to Canton
- Live workshop or demo session at a Canton ecosystem event demonstrating the full developer workflow
- Developer tutorial published on the Canton documentation hub
- Social media and community promotion (Daml forum, Discord)

---

## Motivation

The [2026 Developer Experience Survey](https://discuss.daml.com/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412) is the easiest evidentiary anchor: it ranked local development frameworks the most critical tooling gap and surfaced repeated requests for Foundry-style CLIs. We will not rehash the numbers; the audience for this proposal already knows them. The more interesting question is what changes downstream when a developer can move from a fresh directory to "exercised a choice" in minutes instead of an afternoon.

The first thing that changes is who shows up. Canton has the protocol, the privacy model, and the language. What it has not had until recently is a developer surface that asks for less context up front. A tight inner loop lowers the threshold for the kind of exploratory project — hackathon entries, internal proofs of concept, side experiments — that becomes a real application six months later. Those projects do not ship if the first day is spent stitching together Docker compositions and curl invocations. Each one that does ship is an application on the network, transactions on the Global Synchronizer, and demand for Canton Coin.

The second thing that changes is who stays. Developers who already know Canton spend a measurable fraction of their day on the same friction points: party-ID copy-paste, package-ID lookups, multi-step JSON-API choreography for what should be one shell command. Each of those is an interruption. Removing them is not glamorous work, but the savings compound across a team, across a quarter, across the lifetime of an application.

The third thing that changes is what the AI tooling can do. The same alias-resolution, structured-output, and schema-introspection conventions that make `dforge` pleasant in a shell make it tractable for AI coding agents driving the same workflows. As that integration becomes routine, a clean, machine-friendly CLI as the canonical interaction surface is itself an ecosystem investment.

The wallet commands extend the same logic to Canton's token infrastructure. CIP-56 holdings, FOP transfers, DVP allocations, UTXO consolidation — currently a choice between Canton Console and hand-rolled scripts against the Scan and Registry APIs — become one-line operations. That is the difference between "Canton Coin and CIP-56 tokens are accessible to me" and "I will come back to that next sprint."

---

## Rationale

### Why a CLI Toolkit?

A CLI is composable, cross-platform, and language-agnostic. It can be invoked from a shell, a Makefile, a CI runner, an IDE task, or an AI agent's tool-use loop, with the exact same surface in every case. None of the alternatives — Canton Console, language SDKs, web IDEs — offer that combination, and developers reach for them only when they have to.

Foundry made this pattern the default expectation in Ethereum tooling, and the mapping to Canton is direct:

| Foundry | Daml Forge | Purpose |
|---------|-----------|---------|
| `anvil` | `dforge node` | Local development node |
| `cast` | `dforge call` | Ledger interaction |
| `forge create` / `forge script` | `dforge deploy` | Package deployment |
| `forge build` | `dpm build` (existing) | Compilation |
| `forge test` | `dpm test` (existing) | Unit testing |

The mapping is not coincidental. The build and test cells are deliberately filled by the existing `dpm` commands: those tools are good, the developers building Daml know them, and the Foundation has invested in them. Daml Forge fills the surrounding cells the existing SDK does not cover — interactive ledger exploration, one-command network environments, alias resolution, deployment tooling — and stays out of the cells that are already occupied.

### Why Rust?

The architecture splits along the line of "what the developer sees" versus "what the developer's code targets." The runtime side — JVM, sandbox, Canton protocol — is a settled choice the toolkit does not try to disturb. The CLI side is where every interaction starts and every benchmark of "did this feel fast?" is measured, and there Rust is the right call. A Rust binary starts in ~10 milliseconds with no runtime to load, ships as a single static binary per platform, and parses `.dar` and `.dalf` artifacts natively (via the official protobuf schemas) without a running node. The first matters because developers run these commands hundreds of times per session and notice the difference. The second matters because no one wants to install a JVM to run `dforge call` against a remote testnet. The third matters because static analysis of a compiled package — listing templates, resolving choices, checking dependencies — is a frequent local operation that should not require booting Canton.

The same architectural split made Foundry's `forge` and `cast` workable in Ethereum: native speed where developers feel it, EVM compatibility where it matters. Canton's situation maps cleanly.

### Alternatives Considered

The Dev Fund queue has a healthy collection of adjacent proposals. We considered each as a potential reason to scope this work differently:

- **Extend `dpm` directly.** The Daml Package Manager is rightly focused on build and test. Adding node lifecycle, contract interaction, and alias management would overload its scope and require upstream coordination on every change. Daml Forge complements it from outside without imposing a roadmap on the SDK team.
- **Wrap the Canton Console.** Canton Console is a powerful Scala REPL, and developers who like it should keep using it. Wrapping it in a CLI inherits the JVM startup cost and the Scala syntax, which is the opposite of what the inner-loop ergonomics call for.
- **Multi-node realistic local devnets.** Useful for integration testing of complex topologies. The wrong tool for the inner loop, where the cost of standing the environment up dominates everything else. Daml Forge's single-process sandbox is a deliberate trade in the other direction; multi-node and single-process devnets are complementary, not competing.
- **Static SDK codegen.** Generated typed SDKs are the right answer for application code. They are not the right answer for ad-hoc exploration: the codegen step is friction, the generated surface is verbose, and the developer needs to know the type they want before they can ask for it. Daml Forge bets the other way — runtime alias resolution, no codegen step in the tight loop — and the bet is deliberate. An application built on a generated SDK can still use `dforge` for the moments between writing code.
- **Observability-first toolkits.** Dashboards, log aggregation, and trace viewers are valuable, especially for operators. They are a different problem with a different audience. Daml Forge does not try to compete in that lane.
- **Broad-scope swiss-army CLIs.** A single tool covering scaffolding, package management, testing, deployment, and AI integration is appealing on paper. The trade is breadth versus focus and cost: three small, sharp tools that compose with the rest of the ecosystem are easier to maintain, easier to adopt incrementally, and easier to score against deliverables.
- **Web-based IDE / language SDKs.** Different modalities serving different stages of development. Neither addresses ad-hoc exploration or first-day onboarding, where the friction is highest. No conflict.

### Future Work: Native Daml-LF Runtime

Daml Forge's Rust-based `.dar`/`.dalf` parser — used in `dforge query packages` and template discovery — lays the foundation for a native Daml-LF test runtime. A dev-only Rust interpreter, analogous to Foundry's `revm`, could eliminate the ~5-second JVM startup overhead from the test loop, bringing test execution from seconds to sub-second for typical test suites. This would be scoped exclusively to local development and testing, with relaxed privacy semantics and in-memory-only state, while a conformance test suite ensures behavioral alignment with the production Canton runtime. We envision this as a follow-up proposal building on the tooling and user base established by the current grant.

---

## Long-Term Maintenance

Following delivery of all three milestones:

- **Quarterly compatibility updates** for new Daml SDK and Canton releases
- **Community issue triage** and bug fixes via the open-source repository
- **Open-source under Apache-2.0** in the Canton Foundation GitHub organization — ensuring the project is maintainable by the broader community and not dependent on a single team
- **Documentation updates** aligned with Canton version releases

A maintenance grant proposal may be submitted separately for ongoing support beyond the initial delivery period.
