# GPU/Compiler Knowledge Mine — Synthesis

**Distilled from 38 source documents** | 2026-06-05

---

## Catalog of Every GPU/Compiler Project Attempted

### Projects with Working Implementations

| Project | Status | Language | Key Result |
|---------|--------|----------|------------|
| **depgraph-gpu** | ✅ Working | Rust + CUDA (`cudarc`) | GPU-accelerated dependency graph analysis (topological sort, BFS, SHA-256 hashing) targeting RTX 4050 Ada |
| **flux-vm-gpu** | ✅ Verified | CUDA | FLUX-C constraint VM running on GPU — 321M checks/s sustained, 0/278M+ mismatches |
| **guard2mask compiler** | ✅ Working | Rust | GUARD DSL → FLUX-C bytecode → CUDA kernel → CPU composition pipeline |
| **gpu-aerospace-complex-constraint** | ✅ Verified | CUDA | NOT [30,70] AND ([10,40] OR [60,90]) — 100% correct at 1M scale |
| **gpu-maritime-verification** | ✅ Verified | CUDA | 1M vessels, draft constraint, 24.6% violation rate correctly detected |
| **gpu-boolean-logic-verification** | ✅ Verified | CUDA | AND, OR, combined AND+OR logic — 100% correct across all experiments |
| **gpu-fuzz-200** | ✅ Verified | CUDA | 200 random bytecodes, 20M inputs — zero crashes, zero UB |
| **gpu-end-to-end-pipeline** | ✅ Verified | CUDA | Full GUARD→bytecode→kernel→CPU pipeline, 1M inputs, 100% correct |
| **gpu-multistack-analysis** | ✅ Verified | CUDA | Multi-constraint architecture: separate kernel per constraint, CPU composition |
| **gpu-power-efficiency** | ✅ Verified | CUDA | 63.2M Safe-TOPS/W at 10M batch, 21× improvement from 100K to 10M |
| **gpu-thermal-profile** | ✅ Verified | CUDA | Thermal characterization under sustained load on RTX 4050 |
| **gpu-stress-test** | ✅ Verified | CUDA | Stress testing at maximum throughput |
| **gpu-random-boolean-fuzz** | ✅ Verified | CUDA | Random boolean expression fuzzing against CPU reference |
| **gpu-kernel-robustness-proof** | ✅ Proven | Math | Formal proof: CUDA VM terminates in max_gas steps, no UB possible |
| **gpu-coq-refinement-proof** | ✅ Proven | Coq | CUDA implementation refines Coq operational semantics (stack layout, opcode semantics, gas/fuel correspondence) |

### Projects with Research/Design Documents Only

| Project | Status | Output |
|---------|--------|--------|
| **FLUX constraint-to-ASM compiler** | 📐 Design | Full design for AOT compilation: FLUX constraints → x86-64 assembly directly (20 instructions for 4 constraints, zero dispatch overhead) |
| **SIMD constraint checking (AVX-512)** | 📐 Design | 16 cells simultaneously via AVX-512: ~0.3 cycles/constraint check |
| **LLVM vs Cranelift vs QBE comparison** | 📐 Research | Exhaustive DO-254 DAL A backend comparison — **QBE recommended** (19,700 lines, full formal verification, 12-18 person-month qualification) |
| **compiler-competitive-qwen35b** | 📐 Research | Competitive analysis of compiler strategies |
| **compiler-dx-seed-mini** | 📐 Research | Developer experience and seed design |
| **compiler-engineering-hermes405b** | 📐 Research | Engineering analysis from Hermes 405B model |
| **compiler-formal-deepseek** | 📐 Research | Formal verification approaches |
| **compiler-readme-seed-code** | 📐 Research | README and code documentation strategy |
| **compiler-review-seed-pro** | 📐 Research | Review of compiler architecture decisions |
| **compiler-strategy-qwen397b** | 📐 Research | Strategic compiler development roadmap |
| **compiler-backend-comparison** | 📐 Research | Extended backend comparison for safety-critical systems |
| **flux-vs-llvm-comparison** | 📐 Research | Why FLUX exists independently of LLVM — semantic enforcement vs instruction optimization |
| **flux-vm-formal-verification-analysis** | 📐 Research | Formal verification feasibility for FLUX-C VM |

### Projects Killed or Abandoned

| Project | Why | Lesson |
|---------|-----|--------|
| LLVM as primary backend for safety-critical path | 1.1M lines to audit, no determinism guarantees, 120-180 person-months DAL A qualification | LLVM is an instruction optimizer; FLUX is a semantic enforcer. They serve different purposes |
| Cranelift as primary backend | 142K lines + must qualify rustc (2M more), 17 open miscompilation bugs | "Almost there" technology that stays almost there for 5 years |
| Multi-constraint single-kernel (bytecode composition) | Stack management complexity; separate kernels simpler and already verified | Simplicity wins for safety-critical code |

---

## The FLUX→PTX Architecture Explained

### The Problem

FLUX has a constraint VM with 50 opcodes (stack-based). The performance hierarchy:
- **Python interpreter**: 63M checks/s
- **C interpreter (computed goto)**: 6.15B checks/s (98× Python)
- **GPU (CUDA, verified)**: 321M checks/s sustained, 665M/s peak
- **Theoretical native (compiled)**: ~60B+ checks/s

### The Stack

```
GUARD DSL (human-readable constraints)
    ↓ guard2mask compiler
FLUX-C Bytecode (50 opcodes, stack-based, bounded)
    ↓ per-constraint kernel compilation
CUDA Kernels (one kernel per constraint)
    ↓ parallel execution on GPU
Results → CPU AND/OR composition → Pass/Fail
```

### Key Design Decisions

**1. Stack-based VM with bounded execution.** The FLUX-C VM uses a 64-element stack with gas limiting (max_gas=100). This makes it impossible for the VM to:
- Run forever (gas limit)
- Corrupt memory (bounded stack, no heap)
- Crash on arbitrary input (unknown opcodes are NOP, bounds checked)

The formal proof (`gpu-kernel-robustness-proof`) establishes: for ANY bytecode of ANY length and ANY input, the VM terminates in at most max_gas steps and returns one of {pass, fault, gas-exhausted}. No undefined behavior is possible.

**2. Separate kernel per constraint, CPU composition.** Rather than encoding multi-constraint logic into a single complex kernel, each constraint gets its own kernel launch. The CPU combines results with AND/OR logic. This was verified correct across 156M+ evaluations and is:
- Simpler to implement and debug
- More parallelizable (different constraints can be pipelined)
- Easier to formally verify (each kernel is trivial)

**3. GPU as constraint accelerator, not primary compute.** The GPU excels at batch checking (≥1M inputs per launch, 63.2M Safe-TOPS/W). Below 500K inputs, the CPU scalar path wins. The architecture respects this: GPU for bulk verification, CPU for small batches and composition.

**4. Differential testing as the verification backbone.** Every experiment runs GPU results against a CPU reference implementation. 278M+ total evaluations with zero mismatches. This isn't unit testing — it's continuous equivalence checking.

### The Coq Refinement Bridge

The CUDA implementation is proven to refine the Coq operational semantics through five correspondences:
1. **Stack layout**: Array+SP ↔ Coq list (direct index mapping)
2. **Opcode semantics**: C switch cases ↔ Coq pattern matches (identical behavior)
3. **Gas/Fuel**: CUDA gas counter ↔ Coq fuel parameter (both monotonically decreasing)
4. **Bounded stack safety**: 64-element array ↔ Coq `bounded_stack` lemma
5. **Overall refinement**: Every CUDA transition has a corresponding Coq transition

This is the bridge between "it works on GPU" and "it is formally correct."

### The Backend Decision: QBE, Not LLVM

The exhaustive backend comparison for DO-254 DAL A safety-critical systems concluded:

- **LLVM**: 1,187,000 lines, no determinism guarantees, 120-180 person-months qualification. "The single most common career-ending mistake for safety-critical compiler teams."
- **Cranelift**: 142,000 lines, partial verification, 70-95 person-months (but must qualify rustc). "Almost there technology that stays almost there for 5 years."
- **QBE**: 19,700 lines, full end-to-end Coq verification, absolute determinism, 12-18 person-months qualification. "The boring, unadvertised backend that is purpose-built for exactly this use case."

The 3-phase migration plan (12 months): parallel validation → qualification + shadow deployment → full migration. QBE never reorders instructions, never speculates, produces strictly structured SESE basic blocks, and has zero known miscompilations.

### The AOT Compilation Vision

The constraint-to-assembly design documents a path beyond the GPU VM: compile FLUX constraints directly to native code at constraint-definition time (no JIT needed since constraints are static). Key techniques:

- **Branchless multi-constraint patterns**: 4 constraints in ~12 instructions using `cmov` chains and `adc` (add-with-carry) — zero branches, ~4-6 cycles
- **SIMD batch processing**: AVX-512 processes 16 cells in ~7 instructions (~0.3 cycles/check)
- **Direct register operations**: No dispatch overhead, no opcode decode, no stack manipulation
- **Estimated speedup**: 3-10× scalar over C interpreter, 30-100× with SIMD

---

## Open Problems

### Critical
1. **Theory↔Engine disconnection** — The Lock Algebra proofs (from Oracle1) and the constraint engines (plato-constraints, plato-kernel) don't reference each other. Engine changes could silently invalidate proofs.
2. **Multi-constraint kernel composition** — Current architecture uses separate kernels + CPU composition. A single-kernel approach would reduce latency but needs careful stack management and formal verification.
3. **GPU determinism** — GPU floating-point is not deterministic across runs (different SM scheduling). FLUX uses only integer operations, but the schedulers have CV=0.12 timing jitter. For DAL A, timing determinism matters too.

### High Priority
4. **Cross-language differential testing gap** — The `snap()` bug (36% error rate in Python/JS but correct in Rust) revealed that prototypes and production implementations diverge without systematic cross-checking.
5. **QBE migration path** — The comparison recommends QBE but no implementation exists. The 3-phase plan is documented but not started.
6. **SIMD native compilation** — The AVX-512 and branchless designs are on paper only.

### Research
7. **Self-modifying code for runtime constraint generation** — The design exists (mmap + emit) but no implementation. The question is whether dynamic constraint compilation is needed or if AOT is always sufficient.
8. **FLUX-X (hybrid architecture)** — A superset VM that uses LLVM IR for non-critical tasks while keeping FLUX-C for safety-critical paths. The interface verification between the two layers is estimated at ~10,000 hours.
9. **GPU fleet federation** — Can constraint-checking GPUs across multiple nodes coordinate using the sign-pattern bridge protocol from fleet-architecture?

---

## 5 Recommended Next Steps

### 1. Wire Theory to Engine (1 day)
Add `impl LockAlgebra for PlatoConstraint` trait in `plato-constraints` with doc comments citing specific theorems from Oracle1's proofs. This is the highest-leverage single day of work — it prevents silent proof invalidation.

### 2. Start QBE Backend Prototype (1 sprint)
Implement the Phase 1 translation layer: existing compiler IR → QBE IL (~1100 lines). Run in parallel with the Python generator. Zero production risk. Establishes the 12-month migration clock.

### 3. Build Cross-Language Differential Fuzzer (2-3 days)
Create a test harness that runs the same constraint programs in Python, C, Rust, CUDA, and (future) QBE output. Compare results bit-for-bit. Would have caught the snap() bug in minutes. Target: 100M random programs across all implementations.

### 4. Implement AVX-512 Constraint Kernel (1 sprint)
Take the branchless multi-constraint design from the constraint-to-ASM document and implement it. Target: 4 constraints in ~12 instructions, zero branches. Benchmark against the C interpreter. If the 3-10× speedup materializes, this becomes the production scalar path.

### 5. GPU Federation Prototype (research sprint)
Run two RTX 4050 instances (or simulate on one GPU with two streams) using the sign-pattern bridge protocol from fleet-architecture. Verify that federated constraint checking maintains correctness while scaling throughput. The protocol spec exists (4 bits per agent, 10 bytes/fleet/step) — it needs a GPU implementation.

---

*This synthesis covers 38 source documents totaling ~200KB of research. The GPU/compiler stack is the most experimentally verified component of the SuperInstance ecosystem, with 278M+ constraint evaluations and zero mismatches across all experiments.*
