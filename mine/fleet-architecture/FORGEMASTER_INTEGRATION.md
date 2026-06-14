# Forgemaster + Operational Systems Integration

## Discovery Date: 2026-06-14

The Forgemaster Shell (agent operating protocol) was discovered to be protocol-compatible 
with the existing baton I2I system and GC infrastructure. They were built by different 
workers/instances and never cross-referenced.

## Integration Applied

- `forge-apply.sh` installs forgemaster + wires to GC + registers meta-layer
- `state/.forge/` structure provides cold-start context for forge-protocol agents
- HEARTBEAT.md now includes forge tasks that reference GC and baton systems

## Knowledge Gap Filled

Previously, the forgemaster protocol defined how agents work (commit discipline, 
parallel execution) but had no reference to the operational systems that keep 
the host running. The GC and baton systems defined how the fleet survives but 
had no operating protocol for the agent that manages them.

**Connection**: `forge-apply.sh` — "The forge never cools" now applies to disk 
management and fleet state, not just code. The meta-layer that governs the 
governor.
