# Ternary Fleet Taxonomy v1

## Layer 0: Core & Math (32 crates)
Foundation: Z₃ arithmetic, pure math, theoretical structures

| Crate | Domain | Key Concept |
|-------|--------|-------------|
| ternary-core | Arithmetic | Z₃ add/mul, Trit trait, grids, graphs |
| ternary-types | Types | Ternary enum, TryFrom, Serde |
| ternary-pack | GPU | 2-bit packing, 16× density, XNOR+popcount |
| ternary-field | Algebra | Ternary field operations |
| ternary-ring | Algebra | Ring theory on Z₃ |
| ternary-topology | Topology | Topological spaces on ternary sets |
| ternary-sheaf | Topology | Sheaf cohomology, cellular sheaves |
| ternary-homology | Topology | Simplicial homology, Betti numbers |
| ternary-geometry | Geometry | Geometric algebra on ternary |
| ternary-tensor | Linear Alg | Tensor operations in Z₃ |
| ternary-flux | Analysis | Flux/flow on ternary fields |
| ternary-dynamics | Dynamics | Dynamical systems over Z₃ |
| ternary-hamiltonian | Physics | Hamiltonian mechanics, conservation |
| ternary-noether | Physics | Noether's theorem, symmetry→conservation |
| ternary-electromagnetism | Physics | Maxwell's equations in Z₃ |
| ternary-spiral | Dynamics | Spiral wave dynamics |
| ternary-diehard | Random | Randomness testing (Diehard suite) |
| ternary-genetic | Evolution | Genetic algorithms in Z₃ |
| ternary-complexity | CS Theory | Complexity classes for ternary |
| ternary-entropy | Info Theory | Shannon entropy, information measures |
| ternary-vortex | Fluid | Vortex dynamics |
| ternary-morphogenesis | Biology | Turing patterns, reaction-diffusion |
| ternary-renormalization | Physics | Renormalization group on Z₃ |
| ternary-symmetry | Algebra | Symmetry groups on ternary |
| ternary-morph | Image | Morphological operations (erosion, dilation) |
| ternary-walk | Stochastic | Random walks on Z₃ |
| ternary-collatz | Number Theory | Collatz conjecture in ternary |
| ternary-compiler | PL | AST, bytecode, VM |
| ternary-compiler-optimizer | PL | 6 optimization passes |
| ternary-gauge | Physics | Gauge theory on ternary fields |

## Layer 1: Data Structures & Algorithms (40+ crates)
Classical CS, reimplemented with ternary constraint

| Crate | Domain |
|-------|--------|
| ternary-btree | Trees |
| ternary-heap | Priority queue |
| ternary-sort | Sorting algorithms |
| ternary-hash | Hashing |
| ternary-sketch | Streaming sketches (Count-Min, HLL) |
| ternary-bloom-filter | Approximate membership |
| ternary-search-index | Inverted index, TF-IDF |
| ternary-pagerank | Graph ranking |
| ternary-automata | Finite automata |
| ternary-graph | Graph algorithms |
| ternary-markov | Markov chains |
| ternary-kalman | Kalman filtering |
| ternary-pca | Principal component analysis |
| ternary-reservoir | Reservoir sampling |
| ternary-regex | Regular expressions |
| ternary-interpreter | Bytecode VM |
| ternary-turing | Turing machine |
| ternary-wasm | WASM compilation target |
| ternary-membrane | Membrane computing |

## Layer 2: Distributed Systems (20 crates)
Consensus, coordination, fault tolerance

| Crate | Domain |
|-------|--------|
| ternary-consensus | Consensus protocol |
| ternary-paxos | Paxos with ternary voting |
| ternary-epidemic | Gossip protocol |
| ternary-bloom-filter | Bloom filters |
| ternary-gc | Garbage collection |
| ternary-lease | Lease-based coordination |
| ternary-semaphore | Concurrency control |
| ternary-rate-limiter | Rate limiting |
| ternary-retry | Retry logic |
| ternary-routing | Message routing |
| ternary-version | Vector clocks |
| ternary-antidote | CRDT conflict resolution |
| ternary-priority-queue | Priority scheduling |
| ternary-lattice-gc | Lattice-based GC |
| ternary-backpressure | Flow control |
| ternary-slotmap | Slot allocation |
| ternary-fault-tree | Fault analysis |
| ternary-reassembly | Fragment reassembly |
| ternary-bft | Byzantine fault tolerance |

## Layer 3: ML & Neural (21 crates)
Ternary neural networks, inference, learning

| Crate | Domain |
|-------|--------|
| ternary-attention | Attention mechanism |
| ternary-belief | Belief propagation |
| ternary-classifier | Classification |
| ternary-clustering | Clustering |
| ternary-grad | Gradient descent |
| ternary-inference | Inference engine |
| ternary-tnn | Ternary neural network |
| ternary-llm | LLM kernels |
| ternary-free-energy | Free energy minimization |
| ternary-auto-vectorizer | SIMD vectorization |
| ternary-compress | Model compression |
| ternary-shard | Model sharding |
| ternary-gate | Gating networks |
| ternary-watermark | Watermarking |
| ternary-benchmark | Benchmarking |
| ternary-dispatch | Dispatch routing |

## Layer 4: Applied Systems (21 crates)
Real-world domains where ternary is natural

| Crate | Domain |
|-------|--------|
| ternary-pid | Control systems |
| ternary-thermostat | Climate control |
| ternary-signals | Signal processing |
| ternary-percolate | Percolation theory |
| ternary-budget | Resource allocation |
| ternary-negotiate | Multi-agent negotiation |
| ternary-quorum | Quorum sensing |
| ternary-proof | Proof systems |
| ternary-voting | Voting systems |
| ternary-game-theory | Game theory |
| ternary-blockchain | Blockchain |
| ternary-cipher | Cryptography |
| ternary-fire | Fire spread modeling |
| ternary-chaos | Chaos theory |
| ternary-channel | Channel multiplexing |
| ternary-command | Command dispatch |
| ternary-event | Event sourcing |
| ternary-protocol | Protocol design |
| ternary-scheduler | Task scheduling |
| ternary-cache | Caching |
| ternary-route | Routing |

## Layer 5: Creative & Experimental (17+ crates)
Music, art, novel explorations

| Crate | Domain |
|-------|--------|
| ternary-counterpoint | Music theory |
| ternary-compass | Directional operations |
| ternary-temperament | Musical tuning |
| ternary-rhythm | Rhythm patterns |
| ternary-music | Music generation |
| ternary-polyrhythm | Polyrhythmic patterns |
| ternary-tempo | Tempo tracking |
| ternary-tidelight | Tidal patterns |
| ternary-ear | Ear training |
| ternary-color | Color theory |
| ternary-warp | Warp/teleport |
| ternary-bite | Minimal operations |

## Ports & Cross-Platform (24 crates)

| Crate | Language |
|-------|----------|
| ternary-spreadsheet-c | C |
| ternary-inference-c | C |
| ternary-dissertation-c | C |
| ternary-fitness-c | C |
| ternary-compiler-python | Python |
| ternary-dynamics-python | Python |
| ternary-spreadsheet-python | Python |
| ternary-cell-python | Python |
| ternary-protocol-python | Python |
| ternary-fitness-python | Python |
| ternary-cli | CLI tool |
| ternary-cookbook | Examples collection |

## Cross-Cutting Concerns

### Distributed Coordination Cluster
[gc, lease, semaphore, consensus, paxos, epidemic, version, antidote, backpressure, rate-limiter]

### Ternary Neural Network Stack
[attention, belief, grad, inference, tnn, llm, free-energy, compress, shard, gate, auto-vectorizer]

### Signal Processing Pipeline
[signals, filter, fft, wavelet, sampler, streaming]

### Conservation Law Stack
[hamiltonian, noether, electromagnetism, energy, entropy, flux, vortex]

### Music Theory Stack
[counterpoint, temperament, rhythm, polyrhythm, tempo, ear, color, tidelight]
