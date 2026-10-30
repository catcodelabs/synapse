# Synapse

<p align="center">
  <b>THIS PROJECT IS HIGHLY EXPERIMENTAL AND SHOULD NOT BE USED UNDER ANY CIRCUMSTANCES</b>
</p>
<p align="center">
  <b>THIS PROJECT CURRENTLY DOES NOT CONFORM WITH THE CATCODELABS AI STATEMENT YET</b>
</p>
<p align="center">
  <b>Secure, modular dynamic driver loader and universal telemetric bridge for Monado / OpenXR.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-GPLv3-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/language-Rust-orange.svg" alt="Rust">
  <img src="https://img.shields.io/badge/runtime-Monado%2FOpenXR-green.svg" alt="Monado">
</p>

---

## Philosophy: "Sacrifice the Driver, Protect the Runtime"

**Synapse** is built on a single uncompromising rule: **Monado's core engine and OpenXR server loop must never crash due to third-party code.** 

Instead of forcing developers to write custom C-code or compile proprietary drivers directly into Monado, Synapse introduces a microkernel-style "backdoor" access layer. Heavy networking and data ingestion are safely managed inside a robust Rust host, while untrusted translation and mapping logic run inside a sandboxed scripting environment. If a third-party tracker script throws a fatal error, Synapse isolates it, protects the render loop, and keeps your headset running smoothly.

---

## Architecture & Data Flow

```text
[OSC / UDP on :9595] \
                      ===> [Rust Ingestion Core] ---> [Rhai Translation Map (syn.rhai)] ---> [Monado OpenXR Runtime]
[Unix Domain Socket] /
```

### 1. The Dual-Protocol Ingestion Pipeline

The backend of Synapse runs inside Monado as a high-performance Rust core featuring two non-blocking asynchronous background listeners:
 * OSC / UDP Listener (Port 9595): Provides zero-friction ecosystem compatibility out of the box. Any OSC-based tracking solution can blast data directly into Synapse without any modifications.
 * Unix Domain Sockets (/tmp/synapse.sock): Provides packet-loss-free, stream-oriented performance for custom-built hardware applications.

Both pipelines parse incoming raw bytes asynchronously and cache the latest clean data in an atomic, thread-safe memory pool (Arc<RwLock>), ensuring zero latency or thread-blocking on the main render loop.

### 2. The Sandboxed Scripting Layer (syn.rhai)
Synapse embeds the lightweight, Rust-native Rhai scripting engine to handle translation.
 * Pure Mapping: syn.rhai acts as a clean wiring diagram, pulling raw variables from the host cache and mapping them to standard OpenXR coordinates.
 * No-Panic Guarantee: If a developer makes a syntax or runtime error (such as a division by zero or infinite loop), Rhai traps it securely. Synapse safely disables the offending script and logs the error without harming the runtime.

---

## Developer Workflow & Driver Structure
Drivers are self-contained directories dropped straight into the local user OpenXR configuration path, right next to the /1/ runtime.
```
~/.config/openxr/synapse/<driver_name>.d/
```
### Directory Layout
```
~/.config/openxr/synapse/<driver_name.d/
├── syn.rhai           <-- The standardized mapping script
└── assets/
    └── tracker.obj    <-- 3D visualizer asset manifest
```
### Example syn.rhai Script (example for FluxPose)
```rhai
// 1. Register driver identity and target version contract
syn_register_driver("fluxpose", #{
    syn_version: "1.4.x",
    driver_version: "1.0.0"
});

// 2. Register tracker assets if needed
syn_register_assets([
    "assets/tracker.obj"
    "assets/beacon.obj"
]);

// 3. State initialization (called on startup)
fn on_init() {
    print("[fluxpose] Initialized mapping layer.");
    true
}

// 4. Per-frame poll loop called by Synapse
fn on_poll() {
    // Grab latest cached data from the Rust background listener
    let raw = get_osc_packet("/fluxpose/tracker/1");

    // Map raw array data directly to OpenXR pose properties
    return #{
        type: "positional_tracker",
        position: [raw[0], raw[1], raw[2]],
        orientation: [raw[3], raw[4], raw[5], raw[6]]
    };
}
```
# Key Features
 * Universal Telemetric Bridge: Native OSC/UDP support means you can stream telemetry straight from phone apps, custom microcontrollers, or external tracking solutions.
 * Zero Toolchain Friction: Hardware hobbyists don't need heavy C++ compilation toolchains or memory management mastery; they edit a straightforward, hot-reloadable .rhai script.
 * Fault Isolation: If a script fails, only that specific device feed drops. Monado remains entirely untouched.

---
## License
`synapse` is part of CatCodeLabs, and is therefore licensed under the GNU General Public License v3.0. See `LICENSE` for more information.

