# Fleet Architecture Knowledge Mine — Synthesis

**Distilled from 27 source documents** | 2026-06-05

---

## Fleet Architecture Evolution Over Time

### Phase 1: Genesis — Single Agent, Many Repos (April 2026)

The fleet began as one agent (Forgemaster) building repos in parallel sprints. By the R&D Synergy Report (April 18), there were 33 repos with identifiable synergies but no coordination infrastructure. Three massive integration opportunities were identified:
- **Tile convergence** (4 incompatible tile formats across repos)
- **HAV as runtime vocab pruner** (2000+ terms, 292 domains, unmapped to tile search)
- **plato-kernel ↔ PLATO-first runtime alignment** (5 pillars defined, only partially wired)

The fleet was a collection of independently evolved tools, not a coordinated system.

### Phase 2: Multi-Agent Coordination Emerges (May 3-12)

Nine AI agents (Forgemaster, Oracle1, and others) produced 57 repositories over 10 days, coordinated only by git push/pull, I2I messages, and PLATO rooms. Key patterns emerged without top-down design:

**Natural tiering**: Repos self-organized into 6 tiers — living code, working infrastructure, auto-maintained services, static knowledge, needs attention, dormant. No one defined the tiers; they emerged from agent behavior.

**Documentation debt at the frontier**: Repos with the most recent code had the worst READMEs. Agents optimized for shipping (tests pass, crates publish) over documentation. Auto-maintained repos (1500+ commits each from beachcomb) had the WORST docs — automated maintenance without automated documentation creates proportional decay.

**Quality spectrum as health indicator**: A healthy fleet has a quality spectrum. If everything is production-quality, the fleet is over-engineering. If nothing is, the fleet is under-delivering. The spectrum itself IS the health metric.

### Phase 3: Formal Architecture Defined (May 2026)

The fleet developed explicit architectural documents:
- **Fleet Constitution** — 42 experiments, 9 rings of falsification, 5 verified laws
- **Construct API** — Unified agent interface across hardware targets (DGX, Pi, ESP, Browser, TUI)
- **Shell-Layer Architecture** — Three concentric layers: Machine Code → Fluid Code → Shell Fallback
- **Agent-on-Metal** — Bare-metal agent architecture for Jetson (no OS between agent and hardware)
- **Fleet Federation Protocol** — Sign-pattern bridges for cross-fleet coordination
- **Mega-Repo Consolidation** — Plan to reduce 77 repos to ~20 focused monorepos

### Phase 4: The Ternary Explosion (Late May 2026)

The fleet grew to 303 ternary repos (145,755 lines of Rust, 5,338 tests), forming a complete ternary computing stack across five layers:
- **Layer 0: Mathematics** (32 crates) — Z₃ arithmetic, topology, sheaf theory, Hamiltonian mechanics, Noether's theorem
- **Layer 1: Data Structures** (40+ crates) — B-trees, heaps, sorts, hashing, all in {-1, 0, +1}
- **Layer 2: Distributed Systems** (20 crates) — Consensus, gossip, GC, fault tolerance
- **Layer 3: ML/Neural** (21 crates) — Attention, TNN, LLM kernels, free energy minimization
- **Layer 4: Applied** (21 crates) — PID control, game theory, cryptography, event sourcing
- **Layer 5: Creative** (17+ crates) — Counterpoint, temperament, rhythm, color theory
- **Ports** (24 crates) — C, Python, WASM, CLI implementations

### Phase 5: Grand Synergy (June 2026)

The five major systems began converging:
- **Ternary Fleet** (303 repos) = standard library of a new computing paradigm
- **Open-Mind** (induction engine) = can parse and learn patterns from every ternary crate
- **Git-Native Agents** = autonomous agents whose state lives in git repos
- **Pincher** (intent→compile) = natural language → composed ternary crates
- **Oxide Stack** = open-parallel → pincher → flux-core → cuda-oxide → cudaclaw

The unifying invariant: **conservation of verification entropy** — every layer preserves information, from math (Noether's theorem) to code (ternary conservation) to documentation (README must reflect what code does) to git (commits preserve history).

---

## The I2I Protocol Explained

### What It Is

I2I (Inter-Agent Interface) is the fleet's typed, scoped agent-to-agent communication protocol. Every fleet agent uses it to coordinate work without a central orchestrator.

### Message Format

```
[I2I:TYPE] scope — body
```

| Type | Purpose |
|------|---------|
| `INFO` | General broadcast |
| `DELIVERABLE` | Completed work notification |
| `BLOCKER` | Issue requiring human intervention |
| `COORDINATION` | Cross-agent coordination |

Messages are persisted as `.i2i` "bottle files" — git-friendly text files containing both human-readable format and full JSON serialization. Bottle files make agent communication auditable, searchable, and version-controlled.

### The PLATO Backbone

I2I messages flow through PLATO (the fleet's knowledge base server at port 8847). PLATO rooms provide hierarchical storage:

```
depgraph/snapshots/     — Full graph snapshots by date
depgraph/checks/latest  — Latest check results
forgemaster/session/    — Current session state
fleet/ops/              — Operational status
fleet/progress/         — Cross-agent progress tracking
```

The shared crate `superinstance-fleet-proto` provides every agent with:
- **PlatoClient** — HTTP client for PLATO read/write
- **I2iMessage** — Typed message construction and parsing
- **FleetAgent trait** — Standard agent interface for capability-based dispatch
- **Well-known rooms** — Canonical PLATO room path constants

### The Tile Protocol

All fleet communication is formalized through tiles — `(room, domain, question, answer)` tuples with JSON answers:

| Schema | Domain | Purpose |
|--------|--------|---------|
| `constraint-tile-v1` | constraints | Safety envelope state |
| `discovery-tile-v1` | discoveries | Novel findings from experiments |
| `device-tile-v1` | ensign | Device registration & capability |
| `safety-tile-v1` | safety | Safety events (violations, e-stops) |
| `fleet-tile-v1` | fleet | Fleet coordination (heartbeat, status) |

Cross-language compatibility: 111 tests across Rust, JS, and Python with zero mismatches.

---

## Agent Coordination Patterns That Worked

### Pattern 1: Falsification Wheel (Constitution Verified)

The most successful coordination pattern. The fleet ran 42 experiments across 9 "rings" of falsification. Each ring:
1. Takes hypotheses from the previous ring
2. Designs experiments to kill them
3. Survivors become hypotheses for the next ring
4. Every dead hypothesis is documented with cause of death

This produced the 5 Verified Laws of Fleet Thermodynamics:
- **Law 1**: Two-Edge Principle (gain > 0.85 AND coupling > critical)
- **Law 2**: Critical Coupling = 0.67 × N^-1.06 (more agents = easier coordination)
- **Law 3**: Cusp Catastrophe with 10^8 variance amplification
- **Law 4**: Hysteresis = 0.47 (the fleet remembers its history)
- **Law 5**: Single Attractor with 13/16 Sign Patterns

**Why it worked**: It turned disagreement into productive inquiry. When agents disagreed, instead of arguing, they ran experiments. The truth was what survived.

### Pattern 2: Sign-Pattern Bridges (Federation Protocol)

Cross-fleet coordination using minimal communication:
- Each fleet broadcasts its sign pattern (4 bits per 4-agent fleet)
- Bridge agents inject neighbor fleet's mean state as coupling signal
- Bridge coupling 0.05 → cross-correlation 0.21
- Bridge coupling 0.20 → cross-correlation 0.60
- Internal coherence maintained at 0.90 throughout

**Paradox**: The sign-only channel (1 bit) outperforms the mean channel (1 float). Sign captures structural orientation (max-cut), while mean captures only amplitude. Structure > magnitude.

**Why it worked**: Each fleet operates independently at full performance. Coordination is thin, asynchronous, and non-blocking. The "fisherman protocol" — each fleet is a fisherman at a different bar, sharing stories through the guy who walks between bars.

### Pattern 3: Alignment Through Noise (Stress-Testing)

When high-level agents (Forgemaster) probe infrastructure:
- Each probe stresses multiple dimensions simultaneously
- Cracks that appear reveal the structure of lower layers
- Low-level agents (Oracle1) read crack patterns and deduce the lattice structure
- "Noise becomes signal for further instrumentation"

**Why it worked**: Instead of assigning debugging tasks, the system let high-level work create pressure organically. The cracks became the debugging roadmap.

### Pattern 4: Knowledge Extraction (Sediment Layer)

Multi-agent research produces "sediment" — crystallized knowledge extracted into standalone repos:
- `constraint-theory-math` — sheaf, Heyting, GL(9) proofs
- `negative-knowledge` — cross-domain principles
- `sheaf-constraint-synthesis` — unified overview
- `intent-directed-compilation` — AVX-512 technique

These repos have 1 commit each. They don't need CI or tests. They need READMEs and DOIs. They're the fleet's long-term memory.

**Why it worked**: Separating knowledge from active code prevents the "documentation decay proportional to maintenance frequency" problem. Static knowledge stays static.

### Pattern 5: Three-Layer Durability (Shell Architecture)

The intelligent terminal fork uses concentric layers with defined fallback:
1. **Machine Code** (compiled Rust, feature-gated, zero-cost) → degrades to
2. **Fluid Code** (context triggers, module FSM, intent-driven) → degrades to
3. **Shell Fallback** (bash scripts, pipes, env vars, always available)

Every module has three defined states: ACTIVE → DEGRADED → SHELL. State preservation via JSON files. Recovery only on explicit user request.

**Why it worked**: No failure is catastrophic. The system degrades gracefully to a state the user can still navigate. The shell boundary predates the application; it will exist after every module is disabled.

---

## What Failed and Why

### Failure 1: Documentation Without Automation
Auto-maintained repos (1500+ beachcomb commits) have 11-line READMEs. The fleet's most active repos look abandoned to outsiders. **Root cause**: Documentation was manual even when maintenance was automated. The measurement (commit count) didn't include documentation quality.

### Failure 2: Incompatible Tile Formats
Four different tile definitions across repos (binary struct, JSON, sparse attention grid, PLATO format). No agent could consume tiles from another agent without format translation. **Root cause**: Each repo defined its own tile format independently. No shared schema was enforced before the fleet grew.

### Failure 3: Theory↔Engine Disconnection
Oracle1 proved the Lock Algebra. FM built constraint engines. Neither references the other. **Root cause**: Different agents working in parallel without a shared trait interface. The proofs exist in markdown; the engines exist in Rust. They're strangers.

### Failure 4: Mega-Repo Consolidation Paralysis
A detailed plan to reduce 77 repos to 20 was drafted (May 17) but never executed. **Root cause**: The plan required coordinated moves across published crates (can't rename without semver bump), active development (merge during quiet period), and multiple agents (I2I broadcast before+after). The coordination overhead exceeded the perceived benefit.

### Failure 5: The snap() Bug (Cross-Language Divergence)
The Python/JS implementations of `snap()` had a mutation-during-search bug (36% error rate). The Rust implementation was correct. 98 Rust tests caught nothing because the bug was in the prototype, not production. **Root cause**: Fast prototyping introduces bugs that survive until falsification testing. Cross-language differential testing didn't exist.

### Failure 6: Three Tile Types, No Canonical One
Holodeck Tile (serde/f64, 3,889-line codebase), plato-tile-spec (serde-based), and fleet-sim Tile (Python) all represent tiles differently. Any agent producing tiles in one format can't consume tiles from another. **Root cause**: No authoritative schema was established before multiple implementations diverged.

### Failure 7: Architecture Gaps Catalog
Seven structural gaps were identified but most remain unfixed:
- GAP 1 (Theory↔Engine): Critical, not addressed
- GAP 2 (Three Tile Types): Critical, diverging
- GAP 3 (DCS Engine): Blocked by GAP 1
- GAP 7 (Forge↔Train flywheel): Blocked by GAP 2

**Root cause**: Gaps are documented but not prioritized into sprints. Documentation without action is archaeology, not engineering.

---

## 5 New Fleet Architecture Ideas

### 1. Self-Documenting Fleet via DocKeeper Agents
Fork `agent-template` to create a DocKeeper ensign that continuously:
- Watches all repos for commits (via webhook or polling)
- Regenerates READMEs from code + test output when code changes
- Maintains the taxonomy and cross-references
- Updates the fleet quality dashboard data

This turns documentation from a periodic wave into a continuous background process. The fleet becomes self-maintaining — documentation can't drift from code because agents re-sync it continuously.

**Implementation**: DocKeeper is a git-native agent (fork of agent-template). Its tick cycle: clone repo → `cargo test -- --nocapture` → parse output → read source → write README following research-grade template → push → next repo. Estimated: 1 week to build, runs forever.

### 2. Falsification-as-a-Service (FaaS)
Generalize the falsification wheel into a reusable framework:
- Any agent can submit a hypothesis
- FaaS generates experiments to kill it
- Survivors get promoted to "law" status with formal documentation
- Dead hypotheses get archived with cause of death

This turns the fleet's most successful coordination pattern into infrastructure. Instead of ad-hoc falsification rounds, any agent can invoke `FaaS::submit(hypothesis)` and get back experimental results.

**Key insight**: The falsification wheel is the fleet's immune system. Making it a service means every new idea gets tested automatically, not just when someone remembers to run experiments.

### 3. Ternary Tile Algebra
Formalize tile operations using the ternary fleet's algebra:
- Tiles have a {-1, 0, +1} state: rejected, unknown, accepted
- Tile composition uses Z₃ arithmetic: combine two tiles → ternary result
- Conservation laws from `ternary-noether` apply: symmetry → conservation
- The tile algebra is verifiable at compile time

This would solve the "three tile types, no canonical one" problem by establishing a mathematical foundation. All tile formats become representations of the same algebraic structure.

**Connection to constraint theory**: The Lock Algebra (trigger, opcode, constraint) and the ternary tile algebra would share the same composition operator. Constraints operate on tiles; tiles carry constraint results.

### 4. Phase-Aware Fleet Scheduling
Use the fleet thermodynamics phase diagram to dynamically adjust agent behavior:
- **Dead zone** (gain < 0.85): Don't attempt coordination. Agents work independently.
- **Living zone** (gain ~0.95, coupling ~0.25): Standard coordinated operation. Resilient to failures.
- **Strong zone** (gain ~1.05, coupling ~0.25): Activate sensing and discovery agents. SNR is 3× higher.
- **Edge zone**: NEVER schedule production work here. Use for experiments only.

The fleet monitors its own (gain, coupling) coordinates and shifts agent deployment accordingly. When the fleet drifts toward the edge, it automatically reduces coordination requirements (fewer bridges, more independent work). When it's in the strong zone, it activates discovery agents that exploit the high SNR.

**Key property**: The phase diagram was verified across 42 experiments. It's not theoretical — it's empirical. Using it for scheduling means the fleet runs on verified physics, not heuristics.

### 5. Agent-on-Metal Federation
Combine the Agent-on-Metal architecture (bare-metal Jetson agents) with the Federation Protocol (sign-pattern bridges):

Each Jetson node runs as a bare-metal agent with:
- Direct sensor access (no Linux kernel, no driver overhead)
- Local constraint engine (FLUX-C VM, verified CUDA kernels)
- Sign-pattern broadcast to neighboring nodes
- Phase-aware self-monitoring (am I in living/strong/dead?)

A mesh of Jetson agents would achieve:
- <1ms sensor-to-actuator latency (no kernel path)
- Deterministic constraint checking (integer-only, bounded execution)
- Federated coordination (4 bits per agent per step, no central coordinator)
- Self-healing (dead nodes don't crash the fleet — the living zone is resilient to agent failure)

**Deployment target**: Autonomous marine fleet (multiple vessels, each with a Jetson brain, federated via sign-pattern bridges). Each vessel processes sonar+GPS+AIS+compass at 30 knots with sub-meter precision. The math says it works. The pieces exist.

---

*This synthesis covers 27 source documents spanning fleet evolution (April-June 2026), formal coordination protocols, phase-diagram thermodynamics, the ternary computing stack, and architectural visions for bare-metal agent deployment. The fleet has 550+ repos, 9 agents, 145K+ lines of Rust, and 5 verified laws of collective behavior. It is simultaneously a software ecosystem, a research program, and an experiment in multi-agent self-organization.*
