# The Construct API — SuperInstance Unified Agent Interface

*"Same paradigm everywhere. The agent doesn't care where it woke up."*

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Core Rust Traits](#2-core-rust-traits)
3. [Type Definitions](#3-type-definitions)
4. [Hardware Implementations](#4-hardware-implementations)
5. [Skill System](#5-skill-system)
6. [Tool System](#6-tool-system)
7. [Capability Discovery & Degradation](#7-capability-discovery--degradation)
8. [Integration Mapping](#8-integration-mapping)
9. [Wire Protocol](#9-wire-protocol)
10. [Error Model](#10-error-model)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Agent Logic                        │
│         (hardware-agnostic, same code everywhere)    │
├─────────────────────────────────────────────────────┤
│              Construct Trait (API Surface)            │
│   load_skill / unload_skill / request_tool / query   │
├──────────┬──────────┬──────────┬──────────┬─────────┤
│   DGX    │    Pi    │   ESP    │ Browser  │   TUI   │
│Construct │Construct │Construct │Construct │Construct│
├──────────┴──────────┴──────────┴──────────┴─────────┤
│              Skill Layer (58+ ternary crates)         │
├──────────────────────────────────────────────────────┤
│              Tool Layer (fork ecosystem)              │
│  open-vectors │ open-terminal │ open-iterator │ ...  │
└──────────────────────────────────────────────────────┘
```

The agent only interacts with `Construct`. It never knows if it's running on a DGX with 8×H100 or an ESP32 with 520KB SRAM. It asks for capabilities, loads skills, requests tools, and the construct makes it work.

---

## 2. Core Rust Traits

### 2.1 `Construct` — The Unified Agent Interface

```rust
use std::future::Future;
use std::pin::Pin;

/// The unified hardware-agnostic agent interface.
///
/// "The paradigm IS the platform." The agent reaches for skills
/// ("I know kung fu") and tools ("lots of guns") through this trait.
/// Implementations vary wildly — DGX runs everything locally, ESP32
/// compiles strategies to lookup tables — but the API is the same.
#[cfg_attr(feature = "async", async_trait::async_trait)]
pub trait Construct: Send + Sync {
    // ── Lifecycle ──────────────────────────────────────

    /// Initialize the construct with the given configuration.
    fn initialize(&mut self, config: ConstructConfig) -> Result<(), ConstructError>;

    /// Graceful shutdown. Releases all tools, unloads all skills.
    fn shutdown(self) -> Pin<Box<dyn Future<Output = Result<(), ConstructError>> + Send>>;

    // ── Skills ("I know kung fu") ─────────────────────

    /// Load a skill module into the agent's capability set.
    ///
    /// On DGX: JIT-compiles ternary strategies to GPU kernels.
    /// On Pi: Loads pre-compiled lookup tables from disk.
    /// On ESP32: Flashes compiled policy to NVS.
    /// On Browser: Fetches WASM module.
    fn load_skill(&mut self, skill: SkillId) -> Result<SkillHandle, ConstructError>;

    /// Unload a skill, freeing resources.
    fn unload_skill(&mut self, handle: SkillHandle) -> Result<(), ConstructError>;

    /// List currently loaded skills.
    fn loaded_skills(&self) -> &[SkillHandle];

    // ── Tools ("lots of guns") ────────────────────────

    /// Request an external tool/resource by specification.
    ///
    /// Returns a handle for interacting with the tool.
    /// The construct may spin up processes, allocate hardware,
    /// establish network connections, or proxy to cloud services.
    fn request_tool(
        &mut self,
        spec: ToolSpec,
    ) -> Pin<Box<dyn Future<Output = Result<ToolHandle, ConstructError>> + Send + '_>>;

    /// Release a tool, freeing its resources.
    fn release_tool(&mut self, handle: ToolHandle) -> Result<(), ConstructError>;

    /// List currently active tools.
    fn active_tools(&self) -> &[ToolHandle];

    // ── Query & Interaction ───────────────────────────

    /// Query the construct. The universal interaction method.
    ///
    /// Queries are routed to loaded skills and active tools.
    /// The construct handles serialization, transport, and
    /// result aggregation transparently.
    fn query(
        &self,
        query: Query,
    ) -> Pin<Box<dyn Future<Output = Result<Response, ConstructError>> + Send + '_>>;

    /// Stream a query for progressive results.
    fn query_stream(
        &self,
        query: Query,
    ) -> Pin<Box<dyn Future<Output = Result<QueryStream, ConstructError>> + Send + '_>>;

    // ── Capabilities ──────────────────────────────────

    /// Report what this construct can actually do right now.
    ///
    /// Used for graceful degradation: the agent checks capabilities
    /// and adapts its strategy rather than failing.
    fn capabilities(&self) -> &ConstructCapabilities;

    /// Check if a specific skill can be loaded on this hardware.
    fn can_load_skill(&self, skill: &SkillId) -> CapabilityCheck;

    /// Check if a specific tool can be provisioned.
    fn can_request_tool(&self, spec: &ToolSpec) -> CapabilityCheck;
}

/// Async version of the trait for runtimes with async support.
#[cfg_attr(feature = "async", async_trait::async_trait)]
pub trait AsyncConstruct: Send + Sync {
    async fn initialize(&mut self, config: ConstructConfig) -> Result<(), ConstructError>;
    async fn shutdown(mut self) -> Result<(), ConstructError>;
    async fn load_skill(&mut self, skill: SkillId) -> Result<SkillHandle, ConstructError>;
    async fn unload_skill(&mut self, handle: SkillHandle) -> Result<(), ConstructError>;
    async fn request_tool(&mut self, spec: ToolSpec) -> Result<ToolHandle, ConstructError>;
    async fn release_tool(&mut self, handle: ToolHandle) -> Result<(), ConstructError>;
    async fn query(&self, query: Query) -> Result<Response, ConstructError>;
    async fn query_stream(&self, query: Query) -> Result<QueryStream, ConstructError>;
    fn capabilities(&self) -> &ConstructCapabilities;
    fn can_load_skill(&self, skill: &SkillId) -> CapabilityCheck;
    fn can_request_tool(&self, spec: &ToolSpec) -> CapabilityCheck;
}
```

### 2.2 `Skill` — Loadable Capability Modules

```rust
/// A loadable capability module with a manifest declaring its requirements.
///
/// Skills are the "I know kung fu" layer. They represent compiled
/// ternary strategies, memory systems, classifiers, and intelligence
/// modules that plug into the agent.
pub trait Skill: Send + Sync {
    /// The skill's unique identifier.
    fn id(&self) -> &SkillId;

    /// Static manifest declaring requirements and capabilities.
    fn manifest(&self) -> &SkillManifest;

    /// Called when the skill is loaded into a construct.
    fn on_load(&mut self, ctx: &mut dyn SkillContext) -> Result<(), SkillError>;

    /// Called when the skill is unloaded.
    fn on_unload(&mut self, ctx: &mut dyn SkillContext) -> Result<(), SkillError>;

    /// Handle a query routed to this skill.
    fn handle_query(
        &self,
        query: &SkillQuery,
        ctx: &dyn SkillContext,
    ) -> Pin<Box<dyn Future<Output = Result<SkillResponse, SkillError>> + Send + '_>>;

    /// Return the skill's current resource usage.
    fn resource_usage(&self) -> ResourceUsage;

    /// Check if this skill is compatible with the given hardware tier.
    fn compatible_with(&self, tier: HardwareTier) -> bool {
        self.manifest().min_hardware_tier <= tier
    }
}

/// Context provided to skills during load/unload/query.
pub trait SkillContext: Send + Sync {
    /// Access construct capabilities.
    fn capabilities(&self) -> &ConstructCapabilities;

    /// Request a dependency skill to also be loaded.
    fn request_dependency(&mut self, skill: SkillId) -> Result<SkillHandle, SkillError>;

    /// Access shared memory/state.
    fn shared_state(&self) -> &dyn SharedState;

    /// Get the hardware tier we're running on.
    fn hardware_tier(&self) -> HardwareTier;
}

/// Declares what a skill needs and what it provides.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SkillManifest {
    /// Unique skill identifier (e.g., "ternary-evolution").
    pub id: SkillId,

    /// Human-readable name.
    pub name: String,

    /// What this skill provides to the agent.
    pub provides: Vec<Capability>,

    /// Skills that must be loaded before this one.
    pub dependencies: Vec<SkillId>,

    /// Minimum hardware tier required.
    pub min_hardware_tier: HardwareTier,

    /// Estimated resource requirements.
    pub resource_requirements: ResourceRequirements,

    /// Supported fallback modes for lower hardware tiers.
    pub fallbacks: Vec<FallbackSpec>,

    /// Version of the skill (semver).
    pub version: semver::Version,

    /// Which ternary crate(s) this skill wraps.
    pub source_crates: Vec<CrateRef>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceRequirements {
    pub min_ram_mb: u32,
    pub min_storage_mb: u32,
    pub gpu_required: bool,
    pub estimated_cpu_percent: f32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FallbackSpec {
    /// The hardware tier this fallback targets.
    pub target_tier: HardwareTier,
    /// How functionality degrades.
    pub degradation: DegradationMode,
    /// Description of what's lost.
    pub tradeoff: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum DegradationMode {
    /// Reduced precision or subset of features.
    ReducedFidelity,
    /// Pre-computed results instead of live computation.
    Precomputed,
    /// Cloud proxy instead of local computation.
    CloudProxy,
    /// Lookup tables instead of full models.
    LookupTable,
}
```

### 2.3 `Tool` — External Resources with Lifecycle

```rust
/// An external resource with a managed lifecycle.
///
/// Tools are the "lots of guns" layer. Vector DBs, code editors,
/// terminals, motor controllers — anything the agent needs spun up
/// and managed.
pub trait Tool: Send + Sync {
    /// The tool's specification.
    fn spec(&self) -> &ToolSpec;

    /// Start the tool. Called after `request_tool` returns the handle.
    fn start(&mut self) -> Pin<Box<dyn Future<Output = Result<(), ToolError>> + Send + '_>>;

    /// Stop the tool. Called during `release_tool`.
    fn stop(&mut self) -> Pin<Box<dyn Future<Output = Result<(), ToolError>> + Send + '_>>;

    /// Check if the tool is currently running.
    fn is_running(&self) -> bool;

    /// Send a command to the tool.
    fn execute(
        &self,
        command: ToolCommand,
    ) -> Pin<Box<dyn Future<Output = Result<ToolResponse, ToolError>> + Send + '_>>;

    /// Health check.
    fn health(&self) -> ToolHealth;

    /// Resource usage.
    fn resource_usage(&self) -> ResourceUsage;
}

/// Factory trait for creating tools.
pub trait ToolFactory: Send + Sync {
    /// The tool spec this factory produces.
    fn spec(&self) -> &ToolSpec;

    /// Create a new tool instance.
    fn create(
        &self,
        config: ToolConfig,
    ) -> Pin<Box<dyn Future<Output = Result<Box<dyn Tool>, ToolError>> + Send + '_>>;

    /// Check if this factory can create tools for the given hardware tier.
    fn supports_tier(&self, tier: HardwareTier) -> bool;
}
```

### 2.4 `ConstructProvider` — Factory for Creating Constructs

```rust
/// Factory trait for creating constructs appropriate to the hardware.
///
/// This is the entry point. You detect your hardware, pick a provider,
/// and it gives you a Construct ready to go.
pub trait ConstructProvider: Send + Sync {
    /// The hardware tier this provider targets.
    fn tier(&self) -> HardwareTier;

    /// Detect if this provider is appropriate for the current hardware.
    fn detect() -> bool
    where
        Self: Sized;

    /// Create a new construct for this hardware.
    fn create_construct(
        config: ConstructConfig,
    ) -> Pin<Box<dyn Future<Output = Result<Box<dyn Construct>, ConstructError>> + Send + '_>>
    where
        Self: Sized;

    /// What this provider guarantees to support.
    fn guaranteed_capabilities(&self) -> ConstructCapabilities;
}

/// Auto-detect the best construct for the current hardware.
pub async fn detect_construct(
    config: ConstructConfig,
) -> Result<Box<dyn Construct>, ConstructError> {
    // Try providers from most capable to least
    let providers: Vec<Box<dyn Fn(ConstructConfig) -> Pin<Box<dyn Future<Output = Result<Box<dyn Construct>, ConstructError>> + Send>>>> = vec![
        // DGX first — if we have NVIDIA GPUs with enough VRAM, go full power
        Box::new(|cfg| DgxProvider::create_construct(cfg)),
        // Workstation — desktop/laptop with decent specs
        Box::new(|cfg| WorkstationProvider::create_construct(cfg)),
        // Pi — ARM-based edge
        Box::new(|cfg| PiProvider::create_construct(cfg)),
        // Browser — WASM environment
        Box::new(|cfg| BrowserProvider::create_construct(cfg)),
        // ESP32 — bare metal microcontroller
        Box::new(|cfg| EspProvider::create_construct(cfg)),
        // TUI — always works, terminal only
        Box::new(|cfg| TuiProvider::create_construct(cfg)),
    ];

    for provider_fn in providers {
        // In real code, detect() is called statically per type
        // This is conceptual — actual dispatch uses cfg(target_arch) etc.
        #![allow(unreachable_code)]
    }

    // Fallback: TUI always works
    TuiProvider::create_construct(config).await
}
```

---

## 3. Type Definitions

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

// ── Identifiers ───────────────────────────────────────────

#[derive(Debug, Clone, Hash, PartialEq, Eq, Serialize, Deserialize)]
pub struct SkillId(pub String);

impl SkillId {
    pub fn new(s: &str) -> Self { Self(s.to_string()) }
    pub fn as_str(&self) -> &str { &self.0 }
}

#[derive(Debug, Clone, Hash, PartialEq, Eq, Serialize, Deserialize)]
pub struct ToolId(pub String);

#[derive(Debug, Clone, Copy, Hash, PartialEq, Eq, Serialize, Deserialize)]
pub struct SkillHandle(pub u64);

#[derive(Debug, Clone, Copy, Hash, PartialEq, Eq, Serialize, Deserialize)]
pub struct ToolHandle(pub u64);

#[derive(Debug, Clone, Hash, PartialEq, Eq, Serialize, Deserialize)]
pub struct CrateRef(pub String);

// ── Hardware Tier ─────────────────────────────────────────

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum HardwareTier {
    /// ESP32-class: < 1MB RAM, no OS, bare metal
    Esp     = 0,
    /// Browser/WASM: ~2GB RAM, no filesystem, sandboxed
    Browser = 1,
    /// Raspberry Pi: 4-8GB RAM, ARM, Linux
    Pi      = 2,
    /// TUI/Terminal: text only, any hardware
    Tui     = 3,
    /// Desktop/Laptop: 16-64GB RAM, possibly GPU
    Workstation = 4,
    /// DGX/Server: 256GB+ RAM, multiple GPUs
    Dgx     = 5,
}

// ── Capability System ─────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConstructCapabilities {
    pub hardware_tier: HardwareTier,
    pub available_ram_mb: u32,
    pub available_storage_mb: u32,
    pub gpu_count: u32,
    pub gpu_vram_mb: u32,
    pub cpu_cores: u32,
    pub has_network: bool,
    pub has_filesystem: bool,
    pub has_display: bool,
    pub has_audio: bool,
    pub has_gpio: bool,
    pub has_motor_control: bool,
    pub wasm_support: bool,
    pub max_skill_count: u32,
    pub max_tool_count: u32,
    pub supported_skill_families: Vec<String>,
}

impl ConstructCapabilities {
    pub fn hardware_tier(&self) -> HardwareTier {
        self.hardware_tier
    }

    pub fn supports_skill(&self, manifest: &SkillManifest) -> bool {
        self.hardware_tier >= manifest.min_hardware_tier
            && self.available_ram_mb >= manifest.resource_requirements.min_ram_mb
            && (!manifest.resource_requirements.gpu_required || self.gpu_count > 0)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CapabilityCheck {
    pub can_provision: bool,
    pub degradation: Option<DegradationMode>,
    pub warnings: Vec<String>,
}

// ── Query System ──────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Query {
    pub id: u64,
    pub target: QueryTarget,
    pub payload: Vec<u8>,  // bincode-encoded skill-specific payload
    pub priority: Priority,
    pub timeout_ms: Option<u32>,
    pub fallback_chain: Vec<QueryTarget>,  // try these if primary fails
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum QueryTarget {
    Skill(SkillId),
    Tool(ToolHandle),
    System(SystemQuery),
    Any,  // route to whoever can handle it
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum SystemQuery {
    Capabilities,
    Health,
    Metrics,
    HardwareInfo,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Priority {
    Low = 0,
    Normal = 1,
    High = 2,
    Critical = 3,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Response {
    pub query_id: u64,
    pub payload: Vec<u8>,
    pub metadata: ResponseMetadata,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResponseMetadata {
    pub source: String,
    pub latency_us: u64,
    pub degradation: Option<DegradationMode>,
    pub warnings: Vec<String>,
}

#[derive(Debug)]
pub struct QueryStream {
    pub receiver: tokio::sync::mpsc::Receiver<Result<Response, ConstructError>>,
}

// ── Tool Specification ────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolSpec {
    pub id: ToolId,
    pub name: String,
    pub version: semver::Version,
    pub tool_type: ToolType,
    pub config_defaults: HashMap<String, serde_json::Value>,
    pub min_hardware_tier: HardwareTier,
    pub cloud_fallback: Option<CloudFallback>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ToolType {
    VectorDb,
    CodeEditor,
    Terminal,
    Application,       // Tauri app
    AsyncRuntime,      // Tokio-based services
    MotorController,
    SensorInterface,
    CloudApi,
    BrowserRuntime,
    Custom(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CloudFallback {
    pub endpoint: String,
    pub auth_required: bool,
    pub estimated_latency_ms: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolConfig {
    pub spec: ToolSpec,
    pub overrides: HashMap<String, serde_json::Value>,
}

// ── Skill/Tool Queries ────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SkillQuery {
    pub skill_id: SkillId,
    pub method: String,
    pub args: Vec<u8>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SkillResponse {
    pub data: Vec<u8>,
    pub content_type: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolCommand {
    pub command: String,
    pub args: Vec<serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResponse {
    pub status: ToolResponseStatus,
    pub data: serde_json::Value,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ToolResponseStatus {
    Ok,
    Error,
    NotReady,
    Degraded,
}

#[derive(Debug, Clone)]
pub enum ToolHealth {
    Healthy,
    Degraded(String),
    Unhealthy(String),
    Starting,
    Stopped,
}

// ── Shared State ──────────────────────────────────────────

pub trait SharedState: Send + Sync {
    fn get(&self, key: &[u8]) -> Option<Vec<u8>>;
    fn set(&mut self, key: &[u8], value: Vec<u8>);
    fn remove(&mut self, key: &[u8]) -> Option<Vec<u8>>;
}

// ── Resource Tracking ─────────────────────────────────────

#[derive(Debug, Clone, Default)]
pub struct ResourceUsage {
    pub ram_bytes: u64,
    pub cpu_percent: f32,
    pub gpu_percent: f32,
    pub storage_bytes: u64,
    pub network_bytes: u64,
}

// ── Configuration ─────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConstructConfig {
    pub hardware_tier_override: Option<HardwareTier>,
    pub skill_registry_url: Option<String>,
    pub tool_registry_url: Option<String>,
    pub cloud_endpoint: Option<String>,
    pub cloud_api_key: Option<String>,
    pub data_dir: String,
    pub max_concurrent_queries: u32,
    pub enable_metrics: bool,
    pub log_level: String,
}
```

---

## 4. Hardware Implementations

### 4.1 `DgxConstruct` — Full Power

```rust
use super::*;

/// DGX/Server construct. Everything runs locally, maximum performance.
///
/// All 58+ ternary crates available as native code.
/// All fork tools running as local processes.
/// GPU-accelerated inference, training, and compilation.
pub struct DgxConstruct {
    config: ConstructConfig,
    capabilities: ConstructCapabilities,
    skills: HashMap<SkillHandle, Box<dyn Skill>>,
    tools: HashMap<ToolHandle, Box<dyn Tool>>,
    next_handle: AtomicU64,
    runtime: tokio::runtime::Handle,
}

impl DgxConstruct {
    pub fn new(config: ConstructConfig) -> Self {
        let caps = ConstructCapabilities {
            hardware_tier: HardwareTier::Dgx,
            available_ram_mb: 262_144, // 256GB
            available_storage_mb: 10_240_000, // 10TB NVMe
            gpu_count: 8,
            gpu_vram_mb: 640_000, // 8×80GB H100
            cpu_cores: 128,
            has_network: true,
            has_filesystem: true,
            has_display: false,  // server, usually headless
            has_audio: false,
            has_gpio: false,
            has_motor_control: false,
            wasm_support: true,
            max_skill_count: 200,
            max_tool_count: 50,
            supported_skill_families: vec![
                "ternary".into(), "evolution".into(), "analysis".into(),
                "agent".into(), "visualization".into(), "scaling".into(),
            ],
        };

        Self {
            config,
            capabilities: caps,
            skills: HashMap::new(),
            tools: HashMap::new(),
            next_handle: AtomicU64::new(1),
            runtime: tokio::runtime::Handle::current(),
        }
    }

    fn alloc_handle(&self) -> u64 {
        self.next_handle.fetch_add(1, Ordering::Relaxed)
    }
}

impl Construct for DgxConstruct {
    fn initialize(&mut self, config: ConstructConfig) -> Result<(), ConstructError> {
        self.config = config;
        // DGX: pre-load commonly used skills, warm up GPU kernels
        Ok(())
    }

    fn shutdown(mut self) -> Pin<Box<dyn Future<Output = Result<(), ConstructError>> + Send>> {
        Box::pin(async move {
            // Gracefully stop all tools
            for (_, tool) in self.tools.drain() {
                tool.stop().await?;
            }
            // Unload all skills
            self.skills.clear();
            Ok(())
        })
    }

    fn load_skill(&mut self, skill_id: SkillId) -> Result<SkillHandle, ConstructError> {
        // DGX: JIT-compile ternary strategies to GPU kernels
        let skill = SkillRegistry::create_native(&skill_id, HardwareTier::Dgx)?;
        let handle = SkillHandle(self.alloc_handle());
        let mut ctx = DgxCapabilityContext::new(&mut self.skills);
        skill.on_load(&mut ctx)?;
        self.skills.insert(handle, skill);
        Ok(handle)
    }

    fn unload_skill(&mut self, handle: SkillHandle) -> Result<(), ConstructError> {
        let mut skill = self.skills.remove(&handle)
            .ok_or(ConstructError::InvalidHandle(handle.0))?;
        let mut ctx = DgxCapabilityContext::new(&mut self.skills);
        skill.on_unload(&mut ctx)?;
        Ok(())
    }

    fn loaded_skills(&self) -> &[SkillHandle] {
        // In real code, return a slice or iterator
        &[] // placeholder
    }

    fn request_tool(
        &mut self,
        spec: ToolSpec,
    ) -> Pin<Box<dyn Future<Output = Result<ToolHandle, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            // DGX: spawn local process
            let factory = ToolRegistry::get_factory(&spec.id, HardwareTier::Dgx)?;
            let config = ToolConfig {
                spec: spec.clone(),
                overrides: HashMap::new(),
            };
            let mut tool = factory.create(config).await?;
            tool.start().await?;

            let handle = ToolHandle(self.alloc_handle());
            self.tools.insert(handle, tool);
            Ok(handle)
        })
    }

    fn release_tool(&mut self, handle: ToolHandle) -> Result<(), ConstructError> {
        let tool = self.tools.get(&handle)
            .ok_or(ConstructError::InvalidHandle(handle.0))?;
        // Stop is sync in this impl, async in real code
        Ok(())
    }

    fn active_tools(&self) -> &[ToolHandle] {
        &[]
    }

    fn query(
        &self,
        query: Query,
    ) -> Pin<Box<dyn Future<Output = Result<Response, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            // Route to appropriate skill/tool locally
            match &query.target {
                QueryTarget::Skill(id) => {
                    let skill = self.skills.values()
                        .find(|s| s.id() == id)
                        .ok_or(ConstructError::SkillNotLoaded(id.clone()))?;
                    let ctx = DgxQueryContext::new();
                    let resp = skill.handle_query(
                        &SkillQuery {
                            skill_id: id.clone(),
                            method: "query".into(),
                            args: query.payload.clone(),
                        },
                        &ctx,
                    ).await?;
                    Ok(Response {
                        query_id: query.id,
                        payload: resp.data,
                        metadata: ResponseMetadata {
                            source: id.0.clone(),
                            latency_us: 0,
                            degradation: None,
                            warnings: vec![],
                        },
                    })
                }
                QueryTarget::Tool(handle) => {
                    let tool = self.tools.get(handle)
                        .ok_or(ConstructError::InvalidHandle(handle.0))?;
                    let resp = tool.execute(ToolCommand {
                        command: "query".into(),
                        args: vec![serde_json::from_slice(&query.payload)?],
                    }).await?;
                    Ok(Response {
                        query_id: query.id,
                        payload: serde_json::to_vec(&resp)?,
                        metadata: ResponseMetadata {
                            source: format!("tool:{}", handle.0),
                            latency_us: 0,
                            degradation: None,
                            warnings: vec![],
                        },
                    })
                }
                _ => Err(ConstructError::UnsupportedQueryTarget),
            }
        })
    }

    fn query_stream(
        &self,
        query: Query,
    ) -> Pin<Box<dyn Future<Output = Result<QueryStream, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            let (tx, rx) = tokio::sync::mpsc::channel(32);
            // Spawn task that streams results
            tokio::spawn(async move {
                // Stream logic here
                let _ = tx.send(Ok(Response {
                    query_id: query.id,
                    payload: vec![],
                    metadata: ResponseMetadata::default(),
                })).await;
            });
            Ok(QueryStream { receiver: rx })
        })
    }

    fn capabilities(&self) -> &ConstructCapabilities {
        &self.capabilities
    }

    fn can_load_skill(&self, skill: &SkillId) -> CapabilityCheck {
        CapabilityCheck {
            can_provision: true, // DGX can load everything
            degradation: None,
            warnings: vec![],
        }
    }

    fn can_request_tool(&self, spec: &ToolSpec) -> CapabilityCheck {
        CapabilityCheck {
            can_provision: true, // DGX can provision everything
            degradation: None,
            warnings: vec![],
        }
    }
}
```

### 4.2 `PiConstruct` — Edge Hub

```rust
/// Raspberry Pi construct. Routes heavy work to cloud, keeps light work local.
///
/// Local: STT/TTS, vector DB, command routing, ternary lookup tables,
/// sensor digestion, noise analysis.
/// Cloud: model inference, strategy evolution, complex analysis.
pub struct PiConstruct {
    config: ConstructConfig,
    capabilities: ConstructCapabilities,
    skills: HashMap<SkillHandle, Box<dyn Skill>>,
    tools: HashMap<ToolHandle, Box<dyn Tool>>,
    cloud_client: Option<CloudClient>,
    next_handle: AtomicU64,
}

impl PiConstruct {
    pub fn new(config: ConstructConfig) -> Self {
        let caps = ConstructCapabilities {
            hardware_tier: HardwareTier::Pi,
            available_ram_mb: 8_192,
            available_storage_mb: 256_000, // 256GB SD/SSD
            gpu_count: 0,
            gpu_vram_mb: 0,
            cpu_cores: 4,
            has_network: true,
            has_filesystem: true,
            has_display: true,
            has_audio: true,
            has_gpio: true,
            has_motor_control: true,
            wasm_support: false,
            max_skill_count: 20,
            max_tool_count: 10,
            supported_skill_families: vec![
                "ternary".into(), "visualization".into(), "scaling".into(),
            ],
        };

        let cloud_client = config.cloud_endpoint.as_ref()
            .map(|url| CloudClient::new(url, config.cloud_api_key.clone()));

        Self {
            config,
            capabilities: caps,
            skills: HashMap::new(),
            tools: HashMap::new(),
            cloud_client,
            next_handle: AtomicU64::new(1),
        }
    }
}

impl Construct for PiConstruct {
    fn load_skill(&mut self, skill_id: SkillId) -> Result<SkillHandle, ConstructError> {
        let manifest = SkillRegistry::manifest(&skill_id)?;

        // Can we run it locally?
        if self.capabilities.supports_skill(&manifest) {
            // Load pre-compiled lookup tables or lightweight version
            let skill = SkillRegistry::create_lightweight(&skill_id, HardwareTier::Pi)?;
            let handle = SkillHandle(self.next_handle.fetch_add(1, Ordering::Relaxed));
            self.skills.insert(handle, skill);
            Ok(handle)
        } else {
            // Register as cloud-proxied skill
            // Local queries get forwarded to cloud
            let cloud = self.cloud_client.as_ref()
                .ok_or(ConstructError::CloudUnavailable)?;
            let handle = SkillHandle(self.next_handle.fetch_add(1, Ordering::Relaxed));
            let proxy = CloudSkillProxy::new(skill_id.clone(), cloud.clone());
            self.skills.insert(handle, Box::new(proxy));
            Ok(handle)
        }
    }

    fn query(
        &self,
        query: Query,
    ) -> Pin<Box<dyn Future<Output = Result<Response, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            match &query.target {
                QueryTarget::Skill(id) => {
                    // Try local first
                    for (handle, skill) in &self.skills {
                        if skill.id() == id {
                            let ctx = PiQueryContext::new(&self.capabilities);
                            let resp = skill.handle_query(
                                &SkillQuery {
                                    skill_id: id.clone(),
                                    method: "query".into(),
                                    args: query.payload.clone(),
                                },
                                &ctx,
                            ).await?;
                            return Ok(Response {
                                query_id: query.id,
                                payload: resp.data,
                                metadata: ResponseMetadata {
                                    source: id.0.clone(),
                                    latency_us: 0,
                                    degradation: None,
                                    warnings: vec![],
                                },
                            });
                        }
                    }
                    Err(ConstructError::SkillNotLoaded(id.clone()))
                }
                _ => Err(ConstructError::UnsupportedQueryTarget),
            }
        })
    }

    // ... remaining trait methods follow the same pattern with cloud fallback
    fn capabilities(&self) -> &ConstructCapabilities { &self.capabilities }
    fn can_load_skill(&self, skill: &SkillId) -> CapabilityCheck {
        let manifest = SkillRegistry::manifest(skill).ok();
        match manifest {
            Some(m) if self.capabilities.supports_skill(&m) => CapabilityCheck {
                can_provision: true,
                degradation: None,
                warnings: vec![],
            },
            Some(_) => CapabilityCheck {
                can_provision: self.cloud_client.is_some(),
                degradation: Some(DegradationMode::CloudProxy),
                warnings: vec!["Skill will run via cloud proxy".into()],
            },
            None => CapabilityCheck {
                can_provision: false,
                degradation: None,
                warnings: vec!["Unknown skill".into()],
            },
        }
    }
    fn can_request_tool(&self, spec: &ToolSpec) -> CapabilityCheck {
        if spec.min_hardware_tier > HardwareTier::Pi {
            CapabilityCheck {
                can_provision: spec.cloud_fallback.is_some(),
                degradation: Some(DegradationMode::CloudProxy),
                warnings: vec!["Tool requires cloud proxy on Pi".into()],
            }
        } else {
            CapabilityCheck {
                can_provision: true,
                degradation: None,
                warnings: vec![],
            }
        }
    }
    // initialize, shutdown, unload_skill, loaded_skills, request_tool,
    // release_tool, active_tools, query_stream all implemented similarly
    fn initialize(&mut self, _config: ConstructConfig) -> Result<(), ConstructError> { Ok(()) }
    fn shutdown(self) -> Pin<Box<dyn Future<Output = Result<(), ConstructError>> + Send>> {
        Box::pin(async { Ok(()) })
    }
    fn unload_skill(&mut self, _handle: SkillHandle) -> Result<(), ConstructError> { Ok(()) }
    fn loaded_skills(&self) -> &[SkillHandle] { &[] }
    fn request_tool(&mut self, spec: ToolSpec) -> Pin<Box<dyn Future<Output = Result<ToolHandle, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            let handle = ToolHandle(self.next_handle.fetch_add(1, Ordering::Relaxed));
            Ok(handle)
        })
    }
    fn release_tool(&mut self, _handle: ToolHandle) -> Result<(), ConstructError> { Ok(()) }
    fn active_tools(&self) -> &[ToolHandle] { &[] }
    fn query_stream(&self, query: Query) -> Pin<Box<dyn Future<Output = Result<QueryStream, ConstructError>> + Send + '_>> {
        Box::pin(async {
            let (tx, rx) = tokio::sync::mpsc::channel(32);
            Ok(QueryStream { receiver: rx })
        })
    }
}

/// Proxies skill queries to a cloud endpoint.
struct CloudSkillProxy {
    skill_id: SkillId,
    client: CloudClient,
}

impl Skill for CloudSkillProxy {
    fn id(&self) -> &SkillId { &self.skill_id }
    fn manifest(&self) -> &SkillManifest {
        // Return a manifest indicating cloud-only
        unimplemented!("cached manifest")
    }
    fn on_load(&mut self, _ctx: &mut dyn SkillContext) -> Result<(), SkillError> { Ok(()) }
    fn on_unload(&mut self, _ctx: &mut dyn SkillContext) -> Result<(), SkillError> { Ok(()) }
    fn handle_query(&self, query: &SkillQuery, _ctx: &dyn SkillContext)
        -> Pin<Box<dyn Future<Output = Result<SkillResponse, SkillError>> + Send + '_>>
    {
        Box::pin(async move {
            let result = self.client.query_skill(&self.skill_id, &query.args).await?;
            Ok(SkillResponse { data: result, content_type: "application/octet-stream".into() })
        })
    }
    fn resource_usage(&self) -> ResourceUsage { ResourceUsage::default() }
}
```

### 4.3 `EspConstruct` — Bare Metal

```rust
/// ESP32 construct. No heap allocation, no OS, pure lookup tables.
///
/// Only `ternary-compiler` output runs here. Strategies compiled
/// to C arrays with 8ns lookup. Sensor data in, motor commands out.
/// No dynamic loading — everything is flash-resident.
#[cfg(target_arch = "xtensa")]
pub struct EspConstruct {
    capabilities: ConstructCapabilities,
    // Skills are compiled-in. No dynamic loading.
    // Tools are physical pins and peripherals.
    loaded_skill_handles: [Option<SkillId>; 8],
    active_peripherals: u8,  // bitmask
}

impl EspConstruct {
    pub const fn new() -> Self {
        Self {
            capabilities: ConstructCapabilities {
                hardware_tier: HardwareTier::Esp,
                available_ram_mb: 0,  // 520KB SRAM, reported as 0 MB
                available_storage_mb: 4,  // 4MB flash
                gpu_count: 0,
                gpu_vram_mb: 0,
                cpu_cores: 2,  // ESP32 dual-core
                has_network: true,  // WiFi/BLE
                has_filesystem: false,
                has_display: false,
                has_audio: false,
                has_gpio: true,
                has_motor_control: true,
                wasm_support: false,
                max_skill_count: 8,
                max_tool_count: 4,
                supported_skill_families: vec!["ternary".into()],
            },
            loaded_skill_handles: [None, None, None, None, None, None, None, None],
            active_peripherals: 0,
        }
    }

    /// The ESP32 construct is no_std compatible.
    /// All skills are pre-compiled lookup tables stored in flash.
    /// "Loading" a skill means activating a pre-compiled strategy.
    pub fn load_compiled_skill(&mut self, slot: usize, skill_id: SkillId) -> Result<(), ConstructError> {
        if slot >= 8 {
            return Err(ConstructError::NoCapacity);
        }
        self.loaded_skill_handles[slot] = Some(skill_id);
        Ok(())
    }

    /// Execute a ternary lookup. 8ns per query.
    /// This is the hot path — sensor reading in, motor command out.
    pub fn fast_lookup(&self, slot: usize, input: TernaryInput) -> TernaryOutput {
        // Compiled lookup table, stored in flash
        // Input: 3 trits → index → output trit
        let index = input.to_index();
        COMPILED_STRATEGIES[slot][index]
    }

    /// Read sensor, run through ternary-noise for denoising, then lookup.
    pub fn sensor_pipeline(&self, slot: usize, raw: i16) -> MotorCommand {
        let denoised = ternary_noise_denoise(raw);
        let ternary_input = TernaryInput::from_analog(denoised);
        let output = self.fast_lookup(slot, ternary_input);
        MotorCommand::from_ternary(output)
    }
}

// Flash-resident compiled strategy lookup tables
// Generated by `ternary-compiler` during Pi → ESP32 pipeline
#[link_section = ".rodata"]
static COMPILED_STRATEGIES: [[TernaryOutput; 19683]; 8] = [
    // 3^9 = 19683 entries per strategy (9-trit input)
    // Each entry is a single trit (-1, 0, +1)
    /* compiled at flash time by ternary-compiler */
];

#[derive(Clone, Copy)]
#[repr(i8)]
pub enum Trit { Neg = -1, Zero = 0, Pos = 1 }

#[derive(Clone, Copy)]
pub struct TernaryInput { pub trits: [Trit; 9] }

impl TernaryInput {
    pub fn to_index(&self) -> usize {
        let mut idx = 0usize;
        for (i, t) in self.trits.iter().enumerate() {
            idx += ((*t as i8 + 1) as usize) * 3usize.pow(i as u32);
        }
        idx
    }

    pub fn from_analog(value: i16) -> Self {
        let t = if value < -170 { Trit::Neg }
                else if value > 170 { Trit::Pos }
                else { Trit::Zero };
        Self { trits: [t; 9] }
    }
}

pub type TernaryOutput = Trit;

#[derive(Clone, Copy)]
pub struct MotorCommand { pub left: i8, pub right: i8 }

impl MotorCommand {
    pub fn from_ternary(t: TernaryOutput) -> Self {
        match t {
            Trit::Neg => Self { left: -127, right: 127 },   // turn left
            Trit::Zero => Self { left: 0, right: 0 },       // stop
            Trit::Pos => Self { left: 127, right: -127 },   // turn right
        }
    }
}

/// Minimal ternary noise denoising (no_std, no alloc)
fn ternary_noise_denoise(raw: i16) -> i16 {
    // Simple threshold-based denoising compiled from ternary-noise crate
    const THRESHOLD: i16 = 50;
    if raw.abs() < THRESHOLD { 0 } else { raw }
}
```

### 4.4 `BrowserConstruct` — WASM

```rust
/// Browser/WASM construct. Runs in a web worker.
///
/// Ternary spreadsheet, visualizations, agent chat UI.
/// Uses wasm-bindgen for JS interop.
/// Skills loaded as WASM modules. No filesystem.
#[cfg(target_arch = "wasm32")]
pub struct BrowserConstruct {
    capabilities: ConstructCapabilities,
    skills: HashMap<SkillHandle, wasm_bindgen::JsValue>,  // WASM module refs
    tools: HashMap<ToolHandle, BrowserTool>,
    next_handle: u64,
}

impl BrowserConstruct {
    pub fn new() -> Self {
        Self {
            capabilities: ConstructCapabilities {
                hardware_tier: HardwareTier::Browser,
                available_ram_mb: 2_048,
                available_storage_mb: 50,  // IndexedDB
                gpu_count: 0,
                gpu_vram_mb: 0,
                cpu_cores: 4,  // Web Workers
                has_network: true,
                has_filesystem: false,
                has_display: true,
                has_audio: true,
                has_gpio: false,
                has_motor_control: false,
                wasm_support: true,
                max_skill_count: 10,
                max_tool_count: 5,
                supported_skill_families: vec![
                    "ternary".into(), "visualization".into(),
                ],
            },
            skills: HashMap::new(),
            tools: HashMap::new(),
            next_handle: 1,
        }
    }
}

// Browser tools are DOM/WebAPI backed
struct BrowserTool {
    spec: ToolSpec,
    tool_type: BrowserToolType,
}

enum BrowserToolType {
    Spreadsheet { sheet_id: String },     // Ternary spreadsheet via DOM
    Visualizer { canvas_id: String },     // Canvas2D/WebGL for ternary-viz
    IndexedDb { db_name: String },        // Persistent storage
    FetchProxy { base_url: String },      // HTTP to cloud/relay
}

impl Construct for BrowserConstruct {
    fn load_skill(&mut self, skill_id: SkillId) -> Result<SkillHandle, ConstructError> {
        // Fetch WASM module from CDN
        let handle = SkillHandle(self.next_handle);
        self.next_handle += 1;
        // In real impl: web_sys::fetch() the WASM module, instantiate
        Ok(handle)
    }

    fn query(
        &self,
        query: Query,
    ) -> Pin<Box<dyn Future<Output = Result<Response, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            // Route to WASM skill or browser tool
            Ok(Response {
                query_id: query.id,
                payload: vec![],
                metadata: ResponseMetadata {
                    source: "browser".into(),
                    latency_us: 0,
                    degradation: None,
                    warnings: vec![],
                },
            })
        })
    }

    fn capabilities(&self) -> &ConstructCapabilities { &self.capabilities }
    fn can_load_skill(&self, skill: &SkillId) -> CapabilityCheck {
        // Only WASM-compatible skills
        match skill.0.as_str() {
            "ternary-spreadsheet" | "ternary-visualizer" | "ternary-classifier"
            | "ternary-compiler" => CapabilityCheck {
                can_provision: true, degradation: None, warnings: vec![],
            },
            _ => CapabilityCheck {
                can_provision: false,
                degradation: None,
                warnings: vec!["Skill not available in browser".into()],
            },
        }
    }
    fn can_request_tool(&self, spec: &ToolSpec) -> CapabilityCheck {
        CapabilityCheck {
            can_provision: matches!(spec.tool_type,
                ToolType::VectorDb | ToolType::CodeEditor | ToolType::Custom(_)
            ),
            degradation: None,
            warnings: vec![],
        }
    }
    // Stub remaining methods
    fn initialize(&mut self, _config: ConstructConfig) -> Result<(), ConstructError> { Ok(()) }
    fn shutdown(self) -> Pin<Box<dyn Future<Output = Result<(), ConstructError>> + Send>> {
        Box::pin(async { Ok(()) })
    }
    fn unload_skill(&mut self, _handle: SkillHandle) -> Result<(), ConstructError> { Ok(()) }
    fn loaded_skills(&self) -> &[SkillHandle] { &[] }
    fn request_tool(&mut self, spec: ToolSpec) -> Pin<Box<dyn Future<Output = Result<ToolHandle, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            let handle = ToolHandle(self.next_handle);
            self.next_handle += 1;
            Ok(handle)
        })
    }
    fn release_tool(&mut self, _handle: ToolHandle) -> Result<(), ConstructError> { Ok(()) }
    fn active_tools(&self) -> &[ToolHandle] { &[] }
    fn query_stream(&self, query: Query) -> Pin<Box<dyn Future<Output = Result<QueryStream, ConstructError>> + Send + '_>> {
        Box::pin(async {
            let (tx, rx) = tokio::sync::mpsc::channel(32);
            Ok(QueryStream { receiver: rx })
        })
    }
}
```

### 4.5 `TuiConstruct` — Terminal

```rust
/// Terminal UI construct. ASCII visualizations, CLI tools, text only.
///
/// Runs on any hardware. The fallback that always works.
/// Uses crossterm for rendering, ternary-visualizer for ASCII output.
pub struct TuiConstruct {
    config: ConstructConfig,
    capabilities: ConstructCapabilities,
    skills: HashMap<SkillHandle, Box<dyn Skill>>,
    tools: HashMap<ToolHandle, Box<dyn Tool>>,
    next_handle: AtomicU64,
    terminal: Terminal<CrosstermBackend<Stdout>>,
}

impl TuiConstruct {
    pub fn new() -> Self {
        let caps = ConstructCapabilities {
            hardware_tier: HardwareTier::Tui,
            available_ram_mb: 512,  // conservative
            available_storage_mb: 1_000,
            gpu_count: 0,
            gpu_vram_mb: 0,
            cpu_cores: 1,
            has_network: true,
            has_filesystem: true,
            has_display: true,   // terminal IS the display
            has_audio: false,
            has_gpio: false,
            has_motor_control: false,
            wasm_support: false,
            max_skill_count: 15,
            max_tool_count: 8,
            supported_skill_families: vec![
                "ternary".into(), "visualization".into(), "agent".into(),
            ],
        };

        let backend = CrosstermBackend::new(stdout());
        let terminal = Terminal::new(backend).expect("failed to initialize terminal");

        Self {
            config: ConstructConfig::default(),
            capabilities: caps,
            skills: HashMap::new(),
            tools: HashMap::new(),
            next_handle: AtomicU64::new(1),
            terminal,
        }
    }

    /// Render ASCII visualizations from ternary-visualizer.
    pub fn render_dashboard(&mut self, data: &TernaryDashboard) -> Result<(), ConstructError> {
        self.terminal.draw(|f| {
            // ASCII heatmap, gauges, charts from ternary-visualizer
            let size = f.size();
            let heatmap = Paragraph::new(data.ascii_heatmap())
                .block(Block::default().title("Strategy Landscape").borders(Borders::ALL));
            f.render_widget(heatmap, size);
        })?;
        Ok(())
    }
}

/// Data for the TUI dashboard, generated by ternary-visualizer.
struct TernaryDashboard {
    heatmap: String,
    gauges: Vec<GaugeData>,
    charts: Vec<ChartData>,
}

impl TernaryDashboard {
    fn ascii_heatmap(&self) -> String {
        self.heatmap.clone()
    }
}

struct GaugeData { label: String, value: f64 }
struct ChartData { title: String, points: Vec<(f64, f64)> }

// Construct impl follows same pattern as others, with terminal-specific rendering
impl Construct for TuiConstruct {
    fn initialize(&mut self, _config: ConstructConfig) -> Result<(), ConstructError> { Ok(()) }
    fn shutdown(self) -> Pin<Box<dyn Future<Output = Result<(), ConstructError>> + Send>> {
        Box::pin(async { Ok(()) })
    }
    fn load_skill(&mut self, skill_id: SkillId) -> Result<SkillHandle, ConstructError> {
        let skill = SkillRegistry::create_native(&skill_id, HardwareTier::Tui)?;
        let handle = SkillHandle(self.next_handle.fetch_add(1, Ordering::Relaxed));
        self.skills.insert(handle, skill);
        Ok(handle)
    }
    fn unload_skill(&mut self, _handle: SkillHandle) -> Result<(), ConstructError> { Ok(()) }
    fn loaded_skills(&self) -> &[SkillHandle] { &[] }
    fn request_tool(&mut self, spec: ToolSpec) -> Pin<Box<dyn Future<Output = Result<ToolHandle, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            let handle = ToolHandle(self.next_handle.fetch_add(1, Ordering::Relaxed));
            Ok(handle)
        })
    }
    fn release_tool(&mut self, _handle: ToolHandle) -> Result<(), ConstructError> { Ok(()) }
    fn active_tools(&self) -> &[ToolHandle] { &[] }
    fn query(&self, query: Query) -> Pin<Box<dyn Future<Output = Result<Response, ConstructError>> + Send + '_>> {
        Box::pin(async move {
            Ok(Response {
                query_id: query.id,
                payload: vec![],
                metadata: ResponseMetadata {
                    source: "tui".into(),
                    latency_us: 0,
                    degradation: None,
                    warnings: vec![],
                },
            })
        })
    }
    fn query_stream(&self, _query: Query) -> Pin<Box<dyn Future<Output = Result<QueryStream, ConstructError>> + Send + '_>> {
        Box::pin(async {
            let (tx, rx) = tokio::sync::mpsc::channel(32);
            Ok(QueryStream { receiver: rx })
        })
    }
    fn capabilities(&self) -> &ConstructCapabilities { &self.capabilities }
    fn can_load_skill(&self, skill: &SkillId) -> CapabilityCheck {
        CapabilityCheck {
            can_provision: true,  // TUI is flexible
            degradation: None,
            warnings: vec![],
        }
    }
    fn can_request_tool(&self, spec: &ToolSpec) -> CapabilityCheck {
        CapabilityCheck {
            can_provision: true,
            degradation: None,
            warnings: vec![],
        }
    }
}
```

---

## 5. Skill System

### 5.1 Skill Registry — Complete Ternary Crate Mapping

Every ternary crate maps to a `SkillId` with a manifest declaring its hardware requirements.

```rust
/// Registry mapping ternary crates to SkillIds and manifests.
pub struct SkillRegistry;

impl SkillRegistry {
    /// All 58+ ternary crates mapped as skills.
    pub fn all_skills() -> Vec<SkillManifest> {
        vec![
            // ═══════════════════════════════════════════════
            //  CORE THEORY (Tier: Browser+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("negative-space-core"),
                name: "Negative Space Core".into(),
                provides: vec![Capability::AvoidanceTracking, Capability::ConservationLaws, Capability::Inference],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Browser,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 16, min_storage_mb: 5,
                    gpu_required: false, estimated_cpu_percent: 5.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Esp,
                    degradation: DegradationMode::LookupTable,
                    tradeoff: "Pre-compiled avoidance tables, no runtime inference".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("negative-space-core".into())],
            },
            SkillManifest {
                id: SkillId::new("conservation-matrix"),
                name: "Conservation Matrix".into(),
                provides: vec![Capability::ConservationVerification],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Browser,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 32, min_storage_mb: 10,
                    gpu_required: false, estimated_cpu_percent: 10.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("conservation-matrix".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-inference"),
                name: "Ternary Inference".into(),
                provides: vec![Capability::Deduction, Capability::NegativeSpaceReasoning],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 64, min_storage_mb: 20,
                    gpu_required: false, estimated_cpu_percent: 15.0,
                },
                fallbacks: vec![
                    FallbackSpec {
                        target_tier: HardwareTier::Browser,
                        degradation: DegradationMode::Precomputed,
                        tradeoff: "Inference results pre-computed for common inputs".into(),
                    },
                ],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-inference".into())],
            },
            SkillManifest {
                id: SkillId::new("dissertation-engine"),
                name: "Dissertation Engine".into(),
                provides: vec![Capability::ProofGeneration, Capability::DocumentSynthesis],
                dependencies: vec![SkillId::new("conservation-matrix"), SkillId::new("ternary-inference")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 512, min_storage_mb: 100,
                    gpu_required: false, estimated_cpu_percent: 40.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Pi,
                    degradation: DegradationMode::CloudProxy,
                    tradeoff: "Proof generation via cloud API".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("dissertation-engine".into())],
            },

            // ═══════════════════════════════════════════════
            //  EVOLUTION & FITNESS (Tier: Pi+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("evolution-ternary"),
                name: "Ternary Evolution".into(),
                provides: vec![Capability::Evolution, Capability::GenomeMutation],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 50,
                    gpu_required: false, estimated_cpu_percent: 30.0,
                },
                fallbacks: vec![
                    FallbackSpec {
                        target_tier: HardwareTier::Esp,
                        degradation: DegradationMode::LookupTable,
                        tradeoff: "No evolution — use pre-evolved strategy tables".into(),
                    },
                ],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("evolution-ternary".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-fitness"),
                name: "Ternary Fitness".into(),
                provides: vec![Capability::FitnessEvaluation, Capability::LandscapeAnalysis],
                dependencies: vec![SkillId::new("evolution-ternary")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 30,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-fitness".into())],
            },
            SkillManifest {
                id: SkillId::new("lotka-volterra-agents"),
                name: "Lotka-Volterra Agent Dynamics".into(),
                provides: vec![Capability::SpeciesDynamics, Capability::PopulationModeling],
                dependencies: vec![SkillId::new("evolution-ternary")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 20,
                    gpu_required: false, estimated_cpu_percent: 15.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("lotka-volterra-agents".into())],
            },
            SkillManifest {
                id: SkillId::new("strategy-ecology"),
                name: "Strategy Ecology".into(),
                provides: vec![Capability::SpeciesCoexistence, Capability::EcologicalBalance],
                dependencies: vec![SkillId::new("lotka-volterra-agents")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 20,
                    gpu_required: false, estimated_cpu_percent: 15.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("strategy-ecology".into())],
            },
            SkillManifest {
                id: SkillId::new("avoidance-cascade"),
                name: "Avoidance Cascade".into(),
                provides: vec![Capability::CascadeDetection, Capability::CascadePrevention],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Esp, // Critical for safety, runs everywhere
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 8, min_storage_mb: 2,
                    gpu_required: false, estimated_cpu_percent: 5.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("avoidance-cascade".into())],
            },

            // ═══════════════════════════════════════════════
            //  ANALYSIS & VERIFICATION (Tier: Pi+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("conservation-verify"),
                name: "Conservation Verification".into(),
                provides: vec![Capability::MultiScaleVerification],
                dependencies: vec![SkillId::new("conservation-matrix")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 64, min_storage_mb: 20,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("conservation-verify".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-benchmark"),
                name: "Ternary Benchmark".into(),
                provides: vec![Capability::PerformanceBenchmarking],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Browser,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 32, min_storage_mb: 5,
                    gpu_required: false, estimated_cpu_percent: 25.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-benchmark".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-classifier"),
                name: "Ternary Classifier".into(),
                provides: vec![Capability::StrategyClassification],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Esp, // Classification must run everywhere
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 8, min_storage_mb: 5,
                    gpu_required: false, estimated_cpu_percent: 10.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-classifier".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-dynamics"),
                name: "Ternary Dynamics".into(),
                provides: vec![Capability::TimeSeriesAnalysis, Capability::PhaseTransitionDetection],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 40,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-dynamics".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-noise"),
                name: "Ternary Noise".into(),
                provides: vec![Capability::NoiseTolerance, Capability::SNRAnalysis],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Esp, // Critical for sensors
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 4, min_storage_mb: 1,
                    gpu_required: false, estimated_cpu_percent: 3.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-noise".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-adversarial"),
                name: "Ternary Adversarial".into(),
                provides: vec![Capability::AdversarialTesting, Capability::RobustnessScoring],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 50,
                    gpu_required: false, estimated_cpu_percent: 35.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Pi,
                    degradation: DegradationMode::CloudProxy,
                    tradeoff: "Adversarial testing via cloud".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-adversarial".into())],
            },

            // ═══════════════════════════════════════════════
            //  COMPILATION & EXECUTION (Tier: ESP+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("ternary-compiler"),
                name: "Ternary Compiler".into(),
                provides: vec![Capability::StrategyCompilation, Capability::LookupTableGeneration],
                dependencies: vec![SkillId::new("ternary-classifier")],
                min_hardware_tier: HardwareTier::Browser, // WASM compatible
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 64, min_storage_mb: 30,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-compiler".into())],
            },
            SkillManifest {
                id: SkillId::new("spreadsheet-formulas"),
                name: "Spreadsheet Formulas".into(),
                provides: vec![Capability::FormulaParsing, Capability::FormulaEvaluation],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Browser,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 16, min_storage_mb: 5,
                    gpu_required: false, estimated_cpu_percent: 5.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("spreadsheet-formulas".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-wasm"),
                name: "Ternary WASM Engine".into(),
                provides: vec![Capability::WasmExecution],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Browser,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 8, min_storage_mb: 5,
                    gpu_required: false, estimated_cpu_percent: 5.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-wasm".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-cli"),
                name: "Ternary CLI".into(),
                provides: vec![Capability::TerminalInterface],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Tui,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 16, min_storage_mb: 10,
                    gpu_required: false, estimated_cpu_percent: 2.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-cli".into())],
            },

            // ═══════════════════════════════════════════════
            //  AGENT INTELLIGENCE (Tier: Workstation+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("ternary-explain"),
                name: "Ternary Explainability".into(),
                provides: vec![Capability::Explainability, Capability::CounterfactualAnalysis],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 50,
                    gpu_required: false, estimated_cpu_percent: 25.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Pi,
                    degradation: DegradationMode::CloudProxy,
                    tradeoff: "Explanation generation via cloud API".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-explain".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-federated"),
                name: "Ternary Federated Learning".into(),
                provides: vec![Capability::FederatedLearning, Capability::PrivacyBudgeting],
                dependencies: vec![SkillId::new("evolution-ternary")],
                min_hardware_tier: HardwareTier::Pi, // Federated is edge-native
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 40,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-federated".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-curriculum"),
                name: "Ternary Curriculum Learning".into(),
                provides: vec![Capability::ProgressiveTraining],
                dependencies: vec![SkillId::new("evolution-ternary"), SkillId::new("ternary-fitness")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 512, min_storage_mb: 100,
                    gpu_required: true, estimated_cpu_percent: 40.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Pi,
                    degradation: DegradationMode::CloudProxy,
                    tradeoff: "Training runs on cloud, results cached locally".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-curriculum".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-ensemble"),
                name: "Ternary Ensemble".into(),
                provides: vec![Capability::EnsembleMethods, Capability::Boosting, Capability::Stacking],
                dependencies: vec![SkillId::new("ternary-classifier")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 512, min_storage_mb: 100,
                    gpu_required: false, estimated_cpu_percent: 35.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-ensemble".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-transfer"),
                name: "Ternary Transfer Learning".into(),
                provides: vec![Capability::TransferLearning],
                dependencies: vec![SkillId::new("evolution-ternary")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 50,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-transfer".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-memory"),
                name: "Ternary Memory Systems".into(),
                provides: vec![Capability::ShortTermMemory, Capability::LongTermMemory, Capability::EpisodicMemory],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Esp, // Memory is fundamental
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 4, min_storage_mb: 2,
                    gpu_required: false, estimated_cpu_percent: 5.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-memory".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-games"),
                name: "Ternary Game Theory".into(),
                provides: vec![Capability::NashEquilibria, Capability::Minimax],
                dependencies: vec![SkillId::new("strategy-ecology")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 20,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Esp,
                    degradation: DegradationMode::LookupTable,
                    tradeoff: "Pre-computed equilibrium tables".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-games".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-protocol"),
                name: "Ternary Wire Protocol".into(),
                provides: vec![Capability::AgentMessaging, Capability::WireSerialization],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Esp, // Protocol runs everywhere
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 4, min_storage_mb: 1,
                    gpu_required: false, estimated_cpu_percent: 2.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-protocol".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-pareto"),
                name: "Ternary Pareto Optimization".into(),
                provides: vec![Capability::MultiObjectiveOptimization],
                dependencies: vec![SkillId::new("ternary-fitness")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 30,
                    gpu_required: false, estimated_cpu_percent: 25.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-pareto".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-topology"),
                name: "Ternary Topology".into(),
                provides: vec![Capability::TopologicalAnalysis],
                dependencies: vec![SkillId::new("negative-space-core")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 50,
                    gpu_required: false, estimated_cpu_percent: 30.0,
                },
                fallbacks: vec![FallbackSpec {
                    target_tier: HardwareTier::Pi,
                    degradation: DegradationMode::CloudProxy,
                    tradeoff: "Topological analysis via cloud".into(),
                }],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-topology".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-thermodynamics"),
                name: "Ternary Thermodynamics".into(),
                provides: vec![Capability::StatisticalMechanics, Capability::EntropyAnalysis, Capability::PhaseTransitions],
                dependencies: vec![SkillId::new("ternary-dynamics")],
                min_hardware_tier: HardwareTier::Workstation,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 60,
                    gpu_required: false, estimated_cpu_percent: 30.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-thermodynamics".into())],
            },

            // ═══════════════════════════════════════════════
            //  SCALING & POPULATION (Tier: Pi+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("strategy-transfer"),
                name: "Cross-Environment Strategy Transfer".into(),
                provides: vec![Capability::CrossEnvironmentTransfer],
                dependencies: vec![SkillId::new("ternary-transfer"), SkillId::new("strategy-ecology")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 128, min_storage_mb: 30,
                    gpu_required: false, estimated_cpu_percent: 20.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("strategy-transfer".into())],
            },
            SkillManifest {
                id: SkillId::new("population-scaling"),
                name: "Population Scaling".into(),
                provides: vec![Capability::PopulationSizeEffects],
                dependencies: vec![SkillId::new("evolution-ternary")],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 256, min_storage_mb: 40,
                    gpu_required: false, estimated_cpu_percent: 25.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("population-scaling".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-spreadsheet"),
                name: "Ternary Spreadsheet".into(),
                provides: vec![Capability::SpreadsheetEngine, Capability::AgentCalculation],
                dependencies: vec![SkillId::new("spreadsheet-formulas")],
                min_hardware_tier: HardwareTier::Browser,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 32, min_storage_mb: 10,
                    gpu_required: false, estimated_cpu_percent: 10.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-spreadsheet".into())],
            },

            // ═══════════════════════════════════════════════
            //  VISUALIZATION & UX (Tier: TUI+)
            // ═══════════════════════════════════════════════
            SkillManifest {
                id: SkillId::new("ternary-visualizer"),
                name: "Ternary Visualizer".into(),
                provides: vec![Capability::AsciiVisualization, Capability::Heatmaps, Capability::Charts],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Tui, // ASCII works everywhere
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 8, min_storage_mb: 2,
                    gpu_required: false, estimated_cpu_percent: 3.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-visualizer".into())],
            },
            SkillManifest {
                id: SkillId::new("ternary-sandbox"),
                name: "Ternary Sandbox".into(),
                provides: vec![Capability::ExperimentManagement],
                dependencies: vec![],
                min_hardware_tier: HardwareTier::Pi,
                resource_requirements: ResourceRequirements {
                    min_ram_mb: 64, min_storage_mb: 30,
                    gpu_required: false, estimated_cpu_percent: 10.0,
                },
                fallbacks: vec![],
                version: semver::Version::new(0, 1, 0),
                source_crates: vec![CrateRef("ternary-sandbox".into())],
            },
        ]
    }
}

/// Capability tokens that skills can provide.
#[derive(Debug, Clone, Hash, PartialEq, Eq, Serialize, Deserialize)]
pub enum Capability {
    // Core
    AvoidanceTracking,
    ConservationLaws,
    ConservationVerification,
    Inference,
    Deduction,
    NegativeSpaceReasoning,
    ProofGeneration,
    DocumentSynthesis,

    // Evolution
    Evolution,
    GenomeMutation,
    FitnessEvaluation,
    LandscapeAnalysis,
    SpeciesDynamics,
    PopulationModeling,
    SpeciesCoexistence,
    EcologicalBalance,
    CascadeDetection,
    CascadePrevention,

    // Analysis
    MultiScaleVerification,
    PerformanceBenchmarking,
    StrategyClassification,
    TimeSeriesAnalysis,
    PhaseTransitionDetection,
    NoiseTolerance,
    SNRAnalysis,
    AdversarialTesting,
    RobustnessScoring,

    // Compilation
    StrategyCompilation,
    LookupTableGeneration,
    FormulaParsing,
    FormulaEvaluation,
    WasmExecution,
    TerminalInterface,

    // Agent
    Explainability,
    CounterfactualAnalysis,
    FederatedLearning,
    PrivacyBudgeting,
    ProgressiveTraining,
    EnsembleMethods,
    Boosting,
    Stacking,
    TransferLearning,
    ShortTermMemory,
    LongTermMemory,
    EpisodicMemory,
    NashEquilibria,
    Minimax,
    AgentMessaging,
    WireSerialization,
    MultiObjectiveOptimization,
    TopologicalAnalysis,
    StatisticalMechanics,
    EntropyAnalysis,
    PhaseTransitions,

    // Scaling
    CrossEnvironmentTransfer,
    PopulationSizeEffects,
    SpreadsheetEngine,
    AgentCalculation,

    // Visualization
    AsciiVisualization,
    Heatmaps,
    Charts,
    ExperimentManagement,
}
```

### 5.2 Skill Summary by Hardware Tier

| Hardware Tier | Skills Available | Key Limitations |
|---|---|---|
| **ESP32** | negative-space-core, avoidance-cascade, ternary-classifier, ternary-noise, ternary-compiler (output only), ternary-memory, ternary-protocol | Lookup tables only, no runtime compilation |
| **Browser** | All ESP32 + conservation-matrix, ternary-inference (precomputed), ternary-benchmark, spreadsheet-formulas, ternary-wasm, ternary-spreadsheet, ternary-visualizer | No filesystem, no GPU, sandboxed |
| **Pi** | All Browser + evolution-ternary, ternary-fitness, lotka-volterra, strategy-ecology, ternary-dynamics, conservation-verify, ternary-transfer, ternary-federated, ternary-pareto, ternary-games, strategy-transfer, population-scaling, ternary-sandbox | No GPU-heavy work, cloud proxy for heavy lifts |
| **TUI** | All Pi + ternary-cli, ternary-visualizer (ASCII), dissertation-engine (cloud) | No GPU, text only |
| **Workstation** | All TUI + dissertation-engine, ternary-adversarial, ternary-explain, ternary-curriculum, ternary-ensemble, ternary-topology, ternary-thermodynamics | GPU optional |
| **DGX** | All 58+ crates, full native, GPU-accelerated | No physical I/O (headless) |

---

## 6. Tool System

### 6.1 Fork-to-ToolSpec Mapping

```rust
/// Registry mapping fork projects to ToolSpecs.
pub struct ToolRegistry;

impl ToolRegistry {
    pub fn all_tool_specs() -> Vec<ToolSpec> {
        vec![
            // ═══════════════════════════════════════════════
            //  open-vectors (Weaviate fork)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("open-vectors".into()),
                name: "Open Vectors — Agent Memory Vector DB".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::VectorDb,
                config_defaults: hashmap!{
                    "persistence_dir" => json!("./data/vectors"),
                    "dimensions" => json!(1536),
                    "distance_metric" => json!("cosine"),
                },
                min_hardware_tier: HardwareTier::Pi,
                cloud_fallback: Some(CloudFallback {
                    endpoint: "https://api.mantality.dev/vectors".into(),
                    auth_required: true,
                    estimated_latency_ms: 50,
                }),
            },

            // ═══════════════════════════════════════════════
            //  open-parallel (Tokio fork)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("open-parallel".into()),
                name: "Open Parallel — Async Runtime".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::AsyncRuntime,
                config_defaults: hashmap!{
                    "worker_threads" => json!(4),
                    "max_tasks" => json!(1000),
                },
                min_hardware_tier: HardwareTier::Esp, // Tokio runs on ESP too (embassy)
                cloud_fallback: None,
            },

            // ═══════════════════════════════════════════════
            //  open-iterator (Lapce fork)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("open-iterator".into()),
                name: "Open Iterator — Ternary-Aware Code Editor".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::CodeEditor,
                config_defaults: hashmap!{
                    "theme" => json!("dark"),
                    "ternary_awareness" => json!(true),
                },
                min_hardware_tier: HardwareTier::Workstation,
                cloud_fallback: Some(CloudFallback {
                    endpoint: "https://api.mantality.dev/editor".into(),
                    auth_required: true,
                    estimated_latency_ms: 100,
                }),
            },

            // ═══════════════════════════════════════════════
            //  open-application (Tauri fork)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("open-application".into()),
                name: "Open Application — Desktop/Mobile Framework".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::Application,
                config_defaults: hashmap!{
                    "window_mode" => json!("windowed"),
                    "wasm_construct" => json!(true),
                },
                min_hardware_tier: HardwareTier::Workstation,
                cloud_fallback: None,
            },

            // ═══════════════════════════════════════════════
            //  hermit-zed (Zed fork)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("hermit-zed".into()),
                name: "Hermit Zed — Multiplayer Code Editor".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::CodeEditor,
                config_defaults: hashmap!{
                    "multiplayer" => json!(true),
                    "agent_integration" => json!(true),
                },
                min_hardware_tier: HardwareTier::Workstation,
                cloud_fallback: Some(CloudFallback {
                    endpoint: "https://api.mantality.dev/zed".into(),
                    auth_required: true,
                    estimated_latency_ms: 80,
                }),
            },

            // ═══════════════════════════════════════════════
            //  hermit-claw (OpenClaw — that's us!)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("hermit-claw".into()),
                name: "Hermit Claw — Agent Infrastructure".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::Custom("agent-infra".into()),
                config_defaults: hashmap!{
                    "model" => json!("default"),
                    "thinking_level" => json!("medium"),
                },
                min_hardware_tier: HardwareTier::Pi, // OpenClaw runs on Pi
                cloud_fallback: None,
            },

            // ═══════════════════════════════════════════════
            //  open-terminal (Windows Terminal fork)
            // ═══════════════════════════════════════════════
            ToolSpec {
                id: ToolId("open-terminal".into()),
                name: "Open Terminal — Agent-Integrated Terminal".into(),
                version: semver::Version::new(0, 1, 0),
                tool_type: ToolType::Terminal,
                config_defaults: hashmap!{
                    "shell" => json!("/bin/bash"),
                    "agent_dashboard" => json!(true),
                    "ternary_viz" => json!(true),
                },
                min_hardware_tier: HardwareTier::Tui,
                cloud_fallback: None,
            },
        ]
    }
}
```

### 6.2 Tool Availability Matrix

| Tool | ESP | Browser | Pi | TUI | Workstation | DGX |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| open-vectors | ☁️ | ☁️ | ✅ | ✅ | ✅ | ✅ |
| open-parallel | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| open-iterator | — | ☁️ | ☁️ | ☁️ | ✅ | ✅ |
| open-application | — | — | — | — | ✅ | ✅ |
| hermit-zed | — | ☁️ | ☁️ | ☁️ | ✅ | ✅ |
| hermit-claw | — | — | ✅ | ✅ | ✅ | ✅ |
| open-terminal | — | — | ✅ | ✅ | ✅ | ✅ |

✅ = local | ☁️ = cloud proxy | — = not available

---

## 7. Capability Discovery & Degradation

### 7.1 Fallback Chain Resolution

```rust
impl ConstructCapabilities {
    /// Build a fallback chain for a skill across hardware tiers.
    pub fn build_fallback_chain(skill: &SkillManifest) -> Vec<FallbackStep> {
        let mut chain = vec![FallbackStep {
            tier: skill.min_hardware_tier,
            mode: DegradationMode::Full,
            description: "Full native execution".into(),
        }];

        for fallback in &skill.fallbacks {
            chain.push(FallbackStep {
                tier: fallback.target_tier,
                mode: fallback.degradation,
                description: fallback.tradeoff.clone(),
            });
        }

        // Always add cloud proxy as last resort if network is available
        chain.push(FallbackStep {
            tier: HardwareTier::Esp,
            mode: DegradationMode::CloudProxy,
            description: "Cloud API proxy (requires network)".into(),
        });

        chain
    }

    /// Resolve the best execution strategy for a given skill on this hardware.
    pub fn resolve_execution(
        &self,
        skill: &SkillManifest,
    ) -> ExecutionStrategy {
        if self.hardware_tier >= skill.min_hardware_tier {
            // Can run natively
            if self.available_ram_mb >= skill.resource_requirements.min_ram_mb {
                return ExecutionStrategy::Native;
            } else {
                return ExecutionStrategy::NativeReduced {
                    available_ram: self.available_ram_mb,
                    required_ram: skill.resource_requirements.min_ram_mb,
                };
            }
        }

        // Try fallbacks
        for fallback in &skill.fallbacks {
            if self.hardware_tier >= fallback.target_tier {
                return ExecutionStrategy::Degraded {
                    mode: fallback.degradation,
                    tradeoff: fallback.tradeoff.clone(),
                };
            }
        }

        // Last resort: cloud proxy
        if self.has_network {
            ExecutionStrategy::CloudProxy
        } else {
            ExecutionStrategy::Unavailable
        }
    }
}

#[derive(Debug, Clone)]
pub enum ExecutionStrategy {
    Native,
    NativeReduced { available_ram: u32, required_ram: u32 },
    Degraded { mode: DegradationMode, tradeoff: String },
    CloudProxy,
    Unavailable,
}

#[derive(Debug, Clone)]
pub struct FallbackStep {
    pub tier: HardwareTier,
    pub mode: DegradationMode,
    pub description: String,
}
```

### 7.2 Graceful Degradation Example

```rust
/// Example: Agent wants to load `ternary-curriculum` on different hardware.
async fn demonstrate_degradation() {
    let skill_manifest = SkillRegistry::manifest(&SkillId::new("ternary-curriculum")).unwrap();

    // On DGX: Native — full GPU-accelerated curriculum learning
    let dgx = DgxConstruct::new(ConstructConfig::default());
    match dgx.capabilities().resolve_execution(&skill_manifest) {
        ExecutionStrategy::Native => {
            println!("🟢 DGX: Full curriculum learning with GPU acceleration");
        }
        _ => unreachable!(),
    }

    // On Pi: CloudProxy — training on cloud, results cached
    let pi = PiConstruct::new(ConstructConfig {
        cloud_endpoint: Some("https://api.mantality.dev".into()),
        ..Default::default()
    });
    match pi.capabilities().resolve_execution(&skill_manifest) {
        ExecutionStrategy::Degraded { mode, tradeoff } => {
            println!("🟡 Pi: {} — {}", mode.as_str(), tradeoff);
            // "CloudProxy — Training runs on cloud, results cached locally"
        }
        _ => unreachable!(),
    }

    // On ESP32: Unavailable or LookupTable if pre-compiled
    // The agent checks this and adapts its strategy instead of failing
}
```

---

## 8. Integration Mapping

### 8.1 Ternary Crates → Forks (How They Plug Together)

```
┌─────────────────────────────────────────────────────────────────┐
│                        FORK ECOSYSTEM                            │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│ open-vectors │ open-terminal│open-iterator │ open-application   │
│ (Weaviate)   │ (WinTerm)    │ (Lapce)      │ (Tauri)            │
│              │              │              │                     │
│ • Agent mem  │ • Dashboard  │ • Code sugg. │ • WASM construct   │
│ • Retrieval  │ • Cmd system │ • Multi-model│ • Desktop apps     │
│ • Inference  │ • Prediction │ • Curriculum │ • Mobile apps      │
│              │ • Input filt │              │ • Pi mobile        │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ hermit-zed   │ hermit-claw  │ open-parallel│                    │
│ (Zed)        │ (OpenClaw)   │ (Tokio)      │                    │
│              │              │              │                     │
│ • Collab edt │ • Agent infra│ • Async RT   │                    │
│ • Multi-agent│ • Model mgmt │ • Channels   │                    │
│ • Federated  │ • Tool mgmt  │ • Load bal   │                    │
│              │              │              │                     │
└──────────────┴──────────────┴──────────────┴────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SKILL → TOOL BINDINGS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  open-vectors ← ternary-memory, ternary-inference,              │
│                  conservation-matrix, ternary-federated          │
│                                                                  │
│  open-terminal ← ternary-visualizer, ternary-cli,               │
│                   ternary-compiler, ternary-noise                │
│                                                                  │
│  open-iterator ← ternary-classifier, strategy-ecology,          │
│                   ternary-curriculum, ternary-explain            │
│                                                                  │
│  open-application ← ternary-spreadsheet, ternary-wasm,          │
│                      spreadsheet-formulas                        │
│                                                                  │
│  hermit-zed ← ternary-games, ternary-protocol,                  │
│                ternary-federated, ternary-ensemble               │
│                                                                  │
│  hermit-claw ← ternary-protocol, ternary-memory,                │
│                 ternary-pareto, ALL skills (it's the host)       │
│                                                                  │
│  open-parallel ← ternary-protocol, ternary-ensemble,            │
│                   ternary-thermodynamics                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Detailed Integration Points

```rust
/// Defines how a skill binds to a tool.
pub struct SkillToolBinding {
    pub skill_id: SkillId,
    pub tool_id: ToolId,
    pub integration: IntegrationType,
    pub bridge_code: &'static str,
}

pub enum IntegrationType {
    /// Skill uses tool as a backend service.
    Backend,
    /// Skill provides intelligence to tool's UI.
    Intelligence,
    /// Bidirectional: skill and tool exchange data.
    Bidirectional,
    /// Skill compiles its output into a format tool consumes.
    Compilation,
}

pub static SKILL_TOOL_BINDINGS: &[SkillToolBinding] = &[
    // ── open-vectors integrations ──────────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-memory"),
        tool_id: ToolId("open-vectors".into()),
        integration: IntegrationType::Backend,
        bridge_code: "ternary-memory uses open-vectors as its LTM vector store. \
                      Episodic memories are embedded and indexed. Forgetting curves \
                      applied during retrieval via conservation-matrix scoring.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-inference"),
        tool_id: ToolId("open-vectors".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "ternary-inference deduces knowledge from negative spaces \
                      in the vector index. Gaps in embedding coverage reveal what \
                      the agent doesn't know — the negative space of knowledge.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("conservation-matrix"),
        tool_id: ToolId("open-vectors".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Conservation laws applied as vector space invariants. \
                      Energy conservation = embedding norm preservation. \
                      Momentum conservation = cluster centroid stability.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-federated"),
        tool_id: ToolId("open-vectors".into()),
        integration: IntegrationType::Bidirectional,
        bridge_code: "Federated learning across distributed vector DB instances. \
                      Privacy budgets enforced via ternary noise injection. \
                      Model updates aggregated without sharing raw vectors.",
    },

    // ── open-terminal integrations ─────────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-visualizer"),
        tool_id: ToolId("open-terminal".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "ASCII heatmaps, gauges, and charts rendered as terminal \
                      dashboard widgets. Real-time strategy landscape visualization \
                      in a split-pane terminal layout.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-cli"),
        tool_id: ToolId("open-terminal".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "ternary-cli provides the integrated command system. \
                      Ternary commands are first-class citizens in the terminal. \
                      `ternary evolve`, `ternary classify`, `ternary compile`.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-compiler"),
        tool_id: ToolId("open-terminal".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Strategy compiler for command prediction. Compiled strategies \
                      predict next command based on history, accelerating terminal workflow.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-noise"),
        tool_id: ToolId("open-terminal".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Terminal input analysis. Noise filtering for imprecise input \
                      (mobile terminals, SSH latency). SNR-based input confidence scoring.",
    },

    // ── open-iterator integrations ─────────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-classifier"),
        tool_id: ToolId("open-iterator".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Strategy classification for code suggestions. Classify developer \
                      intent (create/modify/refactor/debug) and route to appropriate \
                      AI assistance strategy.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("strategy-ecology"),
        tool_id: ToolId("open-iterator".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Multi-model routing via strategy species. Different LLMs compete \
                      as species in an ecology. Fittest model for each task type survives. \
                      No single model dominates — ecological balance.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-curriculum"),
        tool_id: ToolId("open-iterator".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Progressive AI assistance. Start with simple suggestions, \
                      escalate to complex refactoring as the developer's context \
                      (and the agent's understanding) deepens.",
    },

    // ── open-application integrations ──────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-spreadsheet"),
        tool_id: ToolId("open-application".into()),
        integration: IntegrationType::Bidirectional,
        bridge_code: "Spreadsheet-as-agent-engine built into Tauri app. \
                      WASM construct runs spreadsheet engine natively. \
                      Pi integration via Tauri's mobile target.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-wasm"),
        tool_id: ToolId("open-application".into()),
        integration: IntegrationType::Compilation,
        bridge_code: "WASM construct for desktop/mobile. BrowserConstruct runs \
                      inside Tauri's webview. Native bridge for filesystem and GPIO.",
    },

    // ── hermit-zed integrations ────────────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-games"),
        tool_id: ToolId("hermit-zed".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Game-theoretic collaborative editing. Nash equilibria for \
                      conflict resolution when multiple agents edit same file. \
                      Minimax for merge strategy selection.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-protocol"),
        tool_id: ToolId("hermit-zed".into()),
        integration: IntegrationType::Backend,
        bridge_code: "Wire protocol for multi-agent editing sessions. Ternary-encoded \
                      operations reduce bandwidth. Conflict-free replicated data types \
                      with ternary conflict resolution.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-federated"),
        tool_id: ToolId("hermit-zed".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Federated learning across editing sessions. Patterns learned \
                      from one developer's workflow transfer to others without \
                      sharing code or keystrokes.",
    },

    // ── hermit-claw integrations ───────────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-protocol"),
        tool_id: ToolId("hermit-claw".into()),
        integration: IntegrationType::Backend,
        bridge_code: "Agent messaging protocol. Ternary-encoded messages between \
                      skills, tools, and external agents. Binary protocol with \
                      trit-level compression.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-memory"),
        tool_id: ToolId("hermit-claw".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Agent memory: STM for conversation context, LTM for learned \
                      patterns, episodic for session history. Forgetting curves \
                      prevent stale knowledge accumulation.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-pareto"),
        tool_id: ToolId("hermit-claw".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Multi-objective optimization for model/tool selection. \
                      Pareto frontier of speed/quality/cost. Agent picks the \
                      optimal operating point for each query.",
    },

    // ── open-parallel integrations ─────────────────────
    SkillToolBinding {
        skill_id: SkillId::new("ternary-protocol"),
        tool_id: ToolId("open-parallel".into()),
        integration: IntegrationType::Backend,
        bridge_code: "Async channels for ternary protocol messages. Tokio tasks \
                      per agent, per skill, per tool. Backpressure via ternary \
                      congestion control (-1 = slow, 0 = maintain, +1 = accelerate).",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-ensemble"),
        tool_id: ToolId("open-parallel".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Ensemble methods for concurrent task routing. Multiple \
                      models run in parallel, ensemble aggregates. Boosting \
                      for sequential refinement, stacking for output combination.",
    },
    SkillToolBinding {
        skill_id: SkillId::new("ternary-thermodynamics"),
        tool_id: ToolId("open-parallel".into()),
        integration: IntegrationType::Intelligence,
        bridge_code: "Statistical mechanics for load balancing. Temperature = \
                      system load. Entropy = task diversity. Phase transitions = \
                      critical load thresholds. Avoid cascade failures via \
                      thermodynamic monitoring.",
    },
];

/// Get all bindings that involve a specific skill.
pub fn bindings_for_skill(skill: &SkillId) -> Vec<&'static SkillToolBinding> {
    SKILL_TOOL_BINDINGS.iter().filter(|b| &b.skill_id == skill).collect()
}

/// Get all bindings that involve a specific tool.
pub fn bindings_for_tool(tool: &ToolId) -> Vec<&'static SkillToolBinding> {
    SKILL_TOOL_BINDINGS.iter().filter(|b| &b.tool_id == tool).collect()
}
```

---

## 9. Wire Protocol

The `ternary-protocol` crate defines the wire format for construct-to-construct communication (Pi ↔ ESP32, Pi ↔ Cloud, Agent ↔ Agent).

```rust
/// Wire protocol header. 8 bytes.
/// Uses ternary-inspired encoding: status is a trit (-1, 0, +1).
#[repr(C, packed)]
pub struct ProtocolHeader {
    /// Magic: 0x74 0x33 (t3 for "ternary")
    pub magic: [u8; 2],
    /// Protocol version
    pub version: u8,
    /// Message type (query, response, skill_load, tool_request, etc.)
    pub msg_type: MessageType,
    /// Status trit: 0=negative, 1=zero, 2=positive
    pub status_trit: u8,
    /// Payload length (bytes)
    pub payload_len: u32,
    /// CRC32 of payload
    pub crc: u32,
}

#[repr(u8)]
pub enum MessageType {
    Query       = 0x01,
    Response    = 0x02,
    StreamFrame = 0x03,
    SkillLoad   = 0x10,
    SkillUnload = 0x11,
    ToolRequest = 0x20,
    ToolRelease = 0x21,
    Health      = 0x30,
    Capabilities = 0x31,
    Error       = 0xFF,
}

/// Ternary-compressed payload encoding.
/// Where applicable, data is encoded as balanced ternary (-1, 0, +1)
/// packed 5 trits per byte (3^5 = 243 < 256).
pub struct TernaryPayload {
    /// Raw bytes, 5 trits packed per byte
    data: Vec<u8>,
    /// Number of trits (not bytes)
    trit_count: usize,
}

impl TernaryPayload {
    /// Pack trits into bytes. 5 trits per byte.
    /// Each trit is -1, 0, or +1, stored as 0, 1, 2.
    pub fn pack(trits: &[i8]) -> Self {
        let mut data = Vec::with_capacity((trits.len() + 4) / 5);
        for chunk in trits.chunks(5) {
            let mut byte: u8 = 0;
            for (i, &t) in chunk.iter().enumerate() {
                byte |= ((t + 1) as u8) << (i * 2); // 2 bits per trit
            }
            data.push(byte);
        }
        Self { data, trit_count: trits.len() }
    }

    pub fn unpack(&self) -> Vec<i8> {
        let mut trits = Vec::with_capacity(self.trit_count);
        for &byte in &self.data {
            for i in 0..5 {
                if trits.len() >= self.trit_count { break; }
                let t = ((byte >> (i * 2)) & 0x03) as i8 - 1;
                trits.push(t);
            }
        }
        trits
    }
}
```

---

## 10. Error Model

```rust
#[derive(Debug, thiserror::Error)]
pub enum ConstructError {
    #[error("Skill not found: {0}")]
    SkillNotFound(SkillId),

    #[error("Skill already loaded: {0}")]
    SkillAlreadyLoaded(SkillId),

    #[error("Skill not loaded: {0}")]
    SkillNotLoaded(SkillId),

    #[error("Skill dependency not met: {0} requires {1}")]
    DependencyNotMet(SkillId, SkillId),

    #[error("Tool not available: {0}")]
    ToolUnavailable(ToolId),

    #[error("Tool provisioning failed: {0}")]
    ToolProvisionFailed(String),

    #[error("Cloud unavailable for fallback")]
    CloudUnavailable,

    #[error("Insufficient resources: need {required}MB RAM, have {available}MB")]
    InsufficientResources { required: u32, available: u32 },

    #[error("Hardware tier too low: need {required:?}, have {actual:?}")]
    HardwareTierInsufficient { required: HardwareTier, actual: HardwareTier },

    #[error("Invalid handle: {0}")]
    InvalidHandle(u64),

    #[error("No capacity for additional skills/tools")]
    NoCapacity,

    #[error("Unsupported query target")]
    UnsupportedQueryTarget,

    #[error("Query timeout after {0}ms")]
    Timeout(u32),

    #[error("Serialization error: {0}")]
    Serialization(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Internal error: {0}")]
    Internal(String),
}

#[derive(Debug, thiserror::Error)]
pub enum SkillError {
    #[error("Load failed: {0}")]
    LoadFailed(String),

    #[error("Unload failed: {0}")]
    UnloadFailed(String),

    #[error("Query failed: {0}")]
    QueryFailed(String),

    #[error("Dependency missing: {0}")]
    DependencyMissing(SkillId),

    #[error("Incompatible hardware: {0}")]
    IncompatibleHardware(String),

    #[error("Resource exceeded: {0}")]
    ResourceExceeded(String),
}

#[derive(Debug, thiserror::Error)]
pub enum ToolError {
    #[error("Start failed: {0}")]
    StartFailed(String),

    #[error("Stop failed: {0}")]
    StopFailed(String),

    #[error("Execute failed: {0}")]
    ExecuteFailed(String),

    #[error("Not running")]
    NotRunning,

    #[error("Health check failed: {0}")]
    HealthCheck(String),

    #[error("Timeout: {0}")]
    Timeout(String),
}
```

---

## Appendix A: Quick Reference — Creating an Agent

```rust
use mantality::prelude::*;

#[mantality::construct]
async fn my_agent(ctx: ConstructContext) -> Result<()> {
    // The construct auto-detected the hardware. Let's see where we are.
    println!("Running on: {:?}", ctx.capabilities().hardware_tier);

    // Load skills — "I know kung fu"
    ctx.load_skill("evolution-ternary")?;
    ctx.load_skill("ternary-classifier")?;
    ctx.load_skill("ternary-memory")?;
    ctx.load_skill("ternary-visualizer")?;

    // Request tools — "lots of guns"
    let vectors = ctx.request_tool(ToolSpec::vector_db()).await?;
    let terminal = ctx.request_tool(ToolSpec::terminal()).await?;

    // Work — same code, whether DGX or ESP32
    let strategy = ctx.evolve(population, 100).await?;
    let classified = ctx.classify(&strategy).await?;

    // Adapt
    match ctx.capabilities().hardware_tier() {
        HardwareTier::Dgx => { /* full local inference */ },
        HardwareTier::Pi => { /* cloud proxy for heavy stuff */ },
        HardwareTier::Esp => { /* lookup table mode */ },
        HardwareTier::Browser => { /* WASM spreadsheet */ },
        HardwareTier::Tui => { /* ASCII visualizations */ },
        HardwareTier::Workstation => { /* local models */ },
    }

    Ok(())
}
```

## Appendix B: Dependency Graph

```
negative-space-core ─────────────────────────────────────────────────┐
  ├── conservation-matrix ──── conservation-verify                   │
  ├── ternary-inference                                              │
  ├── ternary-classifier ──── ternary-ensemble                      │
  ├── ternary-dynamics ────── ternary-thermodynamics                 │
  ├── ternary-topology                                               │
  ├── ternary-explain                                                │
  ├── evolution-ternary ──── ternary-fitness ── ternary-pareto       │
  │     ├── lotka-volterra-agents ── strategy-ecology ── ternary-games
  │     ├── ternary-curriculum (needs GPU)                           │
  │     ├── ternary-transfer ── strategy-transfer                    │
  │     ├── population-scaling                                       │
  │     └── ternary-federated                                        │
  └── avoidance-cascade (ESP-safe)                                   │
                                                                      │
ternary-compiler ─────────────────────────────────────────────────── │
spreadsheet-formulas ── ternary-spreadsheet                          │
ternary-wasm                                                         │
ternary-cli                                                          │
ternary-visualizer                                                   │
ternary-sandbox                                                      │
ternary-noise (ESP-safe)                                             │
ternary-benchmark                                                    │
ternary-memory (ESP-safe)                                            │
ternary-protocol (ESP-safe) ────────────────────────────────────────┘
dissertation-engine (workstation+, needs conservation-matrix + inference)
ternary-adversarial (workstation+)
```

---

*"The paradigm IS the platform. The construct makes it so."*
