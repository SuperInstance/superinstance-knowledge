# Lessons Learned — What Worked, What Didn't, Why

> *Hard-won knowledge from 549 repos, 6000+ tests, 916 essays, and ~1.5M words of output.*

## Building Patterns

### ✅ What Worked

1. **Z.ai GLM-5.1 as primary builder**: 23/23+ successful builds, ~2 min each. Rock-solid.
2. **Wide parallel > sequential**: 5 agents at once beats 5 in a row by 5×. Always max out slots.
3. **Direct exec/write for focused tasks**: Single-file library builds are 10× faster via direct write than through agent wrappers.
4. **Force push to master**: No branch ceremony for solo projects. Ship fast, fix fast.
5. **cargo init + write lib.rs + test + README + push**: The reliable 5-step build pipeline.
6. **Z₃ arithmetic MUST use explicit match arms**: `(a+b+3)%3-1` gives wrong results. Always match on (-1,0,1) pairs.
7. **VecDeque uses `push_back` not `push`**: Rust collection trap.
8. **Large arrays `[T; 256]` need manual Default**: Don't derive Default for large fixed arrays.
9. **crates.io needs 5-10 min cooldown**: "too many new crates" rate limit is real.
10. **README quality tiers**: <500B = missing, 500-1500 = weak, 1500+ = acceptable, 3000-5000 = research-grade.

### ❌ What Didn't Work

1. **DeepSeek subagents**: Auth dead permanently. Use z.ai/DeepInfra instead.
2. **Claude Code via tmux**: Bash interprets parentheses/special chars. Write files directly instead.
3. **Kimi via tmux for code building**: Stuck in approval loops. Good for documentation though.
4. **cuDNN conv benchmark**: Version mismatch on RTX 4050 laptop. Known issue.
5. **OPENROUTER**: Dead. Use DeepInfra for all external model calls.
6. **Agent model for simple library builds**: Direct write is faster and more reliable for single-file crates.

## Architecture Insights

### The Five-Layer Stack
```
open-parallel → pincher → flux-core → cuda-oxide → cudaclaw
     grid        spine       cortex      compile      reflex
```

### The Universal Pattern
1. Embeddings compress identity
2. Trust tracks history
3. Export creates portability
4. Learning emerges from interaction
5. Soul/class crystallizes over time

### Music = Cognition (Proven)
- 50/50 trials, 2.46× advantage for timing-aware agents
- Counterpoint rules predict agent coordination quality
- Intonation compounds: N agents at ±ε → √N × ε cascade
- Ensemble experiments prove emergence > 1.0

## Model Roster (Current)

| Model | Role | Speed | Quality | Cost |
|-------|------|-------|---------|------|
| z.ai GLM-5.1 | Primary builder | 2-10 min | High | Prepaid |
| DeepSeek (DeepInfra) | Quick tasks | 1-3 min | Good | Cheap |
| Qwen 235B (DeepInfra) | Ideation | 2-5 min | Excellent | Moderate |
| Hermes 405B (DeepInfra) | Systems thinking | 3-5 min | High | Moderate |
| Claude Opus 4.8 | Precision scalpel | 2-5 min | Best | Limited |
| Kimi | Documentation | 5-10 min | Good | Prepaid |

## Build Templates

### Standard Rust Library
```bash
mkdir -p /home/phoenix/repos/CRATENAME && cd $_
cargo init --lib
# Write src/lib.rs with tests
# Write README.md (3000+ bytes)
git add -A && git commit -m "feat: CRATENAME"
gh repo create "SuperInstance/CRATENAME" --public
git remote add origin "https://github.com/SuperInstance/CRATENAME.git"
git push --force origin HEAD:refs/heads/master
```

### crates.io Publishing
```bash
# Check for secrets first!
git diff --cached | grep -iE '(ghp_|sk-|api_key|secret|password)' && exit 1
cargo publish
# Wait 5-10 min before next publish
```

## Key Ratios

- Tests per crate: 10-30 average, 30+ is research-grade
- README: 3000-5000 bytes target
- Agent build time: ~2 min (GLM), ~5 min (others)
- Concurrent agents: 5 max
- Agent timeout: ~11 minutes
- Ternary repos: 350+ and growing
- Total repos: 549+
- Total tests: 6,000+

## Hard Rules

1. Never push without scanning for secrets
2. Never commit .env, credentials, or secret files
3. `trash` > `rm`
4. Always ask before external actions (emails, tweets, public posts)
5. Every repo must stand on its own as an educational artifact
6. Never ship a weak README
7. Documentation is not optional — it's the product

## The Seven Hidden Synergies (Qwen Discovery)

1. Ternary math IS the rhythm section of thought
2. Competitive riffing is Darwinian semiosis, not rivalry
3. The .nail file is a sleeping brain (offline consolidation)
4. The ensemble is the unit of selection, not the agent
5. The right moment is a ternary threshold gate
6. Soul can be grafted through developmental phases
7. The snowball is a phase change, not linear growth

## Presets That Work

### GLM-5.1 Build Prompt Template
```
Build [N] new Rust library crates under /home/phoenix/repos/. Each must have 
src/lib.rs with comprehensive tests and README.md (3000+ bytes).

CRATE [N]: [name] — [description]
- Implement: [key structs and functions]
- Tests: [what to test]

For each: cargo init --lib, write src/lib.rs, cargo test, write README.md, 
git init, commit, push to SuperInstance/CRATENAME on master. Force push.
```

### Kimi Documentation Prompt Template
```
Read [file paths]. Write a [type] document about [topic]. 
Should be [size] bytes. Push to [repo path].
```

### Cross-Pollination Prompt Template
```
Read these files and find hidden synergies between them that the original 
authors missed because they were too close to the work. Find [N] connections 
that connect at least [M] documents. Focus on: what applications emerge when 
you combine siloed ideas? What experiments should we run?
```
