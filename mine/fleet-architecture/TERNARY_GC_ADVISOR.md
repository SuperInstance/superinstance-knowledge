# Ternary GC Advisor

## Discovery/Connection Date: 2026-06-14

A lightweight Python swarm (9 particles, 4 policy axes, {-1,0,+1} grid) that
sits on top of gc-intelligent.sh and recommends PID parameters.

### Innovation
This connects three previously independent creations:
1. **ternary-swarm** (Rust crate) — provided the concept of particles voting on
   a ternary grid
2. **cocapn** (Python agent) — provides the tile memory/flywheel for learning
3. **gc-pid-bridge** (Rust binary) — provides the PID that the advisor tunes

### Integration
- `scripts/ternary-gc-advisor.py` — 350 lines, pure Python, no deps
- Auto-detected by gc-intelligent.sh on each cycle
- Swarm converges on GC policy from real ledger data (47+ entries)
- Currently recommending: aggressive (10%) setpoint with narrow deadband

### Why this matters
The GC was already self-aware (it examines its own ledger). The swarm advisor
makes it *self-governing* — the policy is derived from the data, not hardcoded.
This is the same ternary decision theory applied at a meta-layer: the GC watches
the disk, the advisor watches the GC, and the ternary math is the same at
every level.
