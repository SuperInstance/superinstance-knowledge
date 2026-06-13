# Cross-Domain Synergy Discovery

*February — note generated June 2026*

## Discovery

Three independent implementations of ternary control were identified in the fleet:

| System | Domain | Technology | Status |
|--------|--------|-----------|--------|
| `gc-intelligent.sh` | Host disk management | Bash + `bc` PID | Live on Oracle2 |
| `ternary-pid` | Process control / actuation | Rust crate | Published |
| `ternary-gc` | GPU memory management | Rust crate | Published |

These share the same mathematics but don't reference each other. This is a **missed cross-pollination opportunity**.

## The Isomorphism

All three are applications of:

$$\text{State} \in \{-1, 0, +1\}$$
$$\text{Control} = f(\text{setpoint}, \text{measurement}) \in \{-1, 0, +1\}$$

Where the {−1, 0, +1} maps to:

| System | −1 | 0 | +1 |
|--------|----|---|----|
| gc-intelligent | Evict aggressively | Leave alone | Keep at high cost |
| ternary-pid | Pull backward | Do nothing | Push forward |
| ternary-gc | Unreachable | Maybe-reachable | Reachable |

## Fleet Architecture Impact

The grand synergy identifies a 5-layer oxide stack (intent → pincher → flux → CUDA-oxide → GPU). The GC system operates at a *sixth* layer below everything: **metal/host**. This layer is invisible to the stack architecture docs but critical for operations.

### Proposed layer structure:

```
Layer 6: Application    — cocapn, spreadsheets, fleet agents
Layer 5: Compile/Intents — pincher
Layer 4: Bytecode        — flux-core
Layer 3: GPU Runtime     — cuda-oxide / ternary-gc
Layer 2: GPU Metal       — cudaclaw (PTX)
Layer 1: Host Metal      — gc-intelligent (disk, memory, process)  ← MISSING FROM STACK DOCS
Layer 0: Hardware        — ARM64, x86_64, GPU
```

### Cascade control across layers

When the host GC evicts a GPU process's cache files, the GPU GC (`ternary-gc`) should be notified to mark those objects as `MaybeReachable` (0). This is a cascade PID architecture:

```
Host GC (disk pressure) → setpoint for GPU GC → setpoint for process control → ternary actuation
```

No such communication channel exists yet.

## Updated Knowledge Graph

```
superinstance-knowledge: the theory
    ↕ cross-references ↑
baton-system: the operational host GC (gc-intelligent.sh)
    ↕ cross-references ↑
ternary-gc: GPU memory GC (Rust)
    ↕ cross-references ↑
ternary-pid: Ternary control (Rust)
    ↕ cross-references ↑
gc-pid-bridge: (PROPOSED) Rust wrapper for gc-intelligent.sh 
```

## References

- `baton-system/docs/CROSS_DOMAIN_SYNERGY.md` — full analysis
- `baton-system/docs/GC_AGENTS.md` — fleet GC specification
- `baton-system/docs/gc-intelligent-README.md` — host-level GC system
- `ternary-gc/src/lib.rs` — ternary mark-sweep GC
- `ternary-pid/src/lib.rs` — ternary PID controller
- `mine/fleet-architecture/GRAND_SYNERGY.md` — the grand vision
