# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Chipyard

Chipyard is a framework for agile development of Chisel-based RISC-V SoCs. It integrates processor cores (Rocket, BOOM, CVA6), accelerators (Gemmini, NVDLA), vector units (Saturn, Ara), and provides simulation, FPGA, and VLSI flows.

## Environment Setup

```bash
# Source environment (activates conda env, sets RISCV, etc.)
source env.sh

# Full initial setup (conda, submodules, toolchains, precompilation)
./build-setup.sh
```

## Build & Simulation Commands

All simulation commands run from `sims/verilator/` (or `sims/vcs/`). The build system uses SBT for Scala compilation and Make for RTL/simulation flow.

```bash
# Generate Verilog (default: RocketConfig)
make -C sims/verilator verilog

# Generate Verilog for a specific config
make -C sims/verilator CONFIG=RocketConfig verilog
make -C sims/verilator CONFIG=LargeBoomV3Config verilog

# Build Verilator simulator
make -C sims/verilator

# Build debug simulator (with waveform support)
make -C sims/verilator debug

# Run a binary on the simulator
make -C sims/verilator run-binary BINARY=<path-to-elf>
make -C sims/verilator run-binary-fast BINARY=<path-to-elf>       # no instruction log
make -C sims/verilator run-binary-debug BINARY=<path-to-elf>      # with waveform dump

# Run assembly/benchmark tests
make -C sims/verilator run-tests

# Clean build artifacts
make -C sims/verilator clean

# Launch SBT shell for interactive Scala development
make -C sims/verilator launch-sbt

# List available configs
make -C sims/verilator find-configs

# List all config fragments
make -C sims/verilator find-config-fragments
```

**Key Make variables:**
- `CONFIG` - Configuration class name (default: `RocketConfig`)
- `CONFIG_PACKAGE` - Scala package for CONFIG (default: `chipyard`)
- `SUB_PROJECT` - Subproject preset: `chipyard`, `testchipip`, `rocketchip`, `constellation`
- `BINARY` - ELF binary path for simulation
- `VERILATOR_THREADS` - Simulator thread count (default: 1)
- `USE_FST=1` - Use FST waveform format instead of VCD
- `USE_CHISEL7=1` - Build with Chisel 7 (experimental)

## SBT Commands

```bash
# From repo root
java -jar scripts/sbt-launch.jar

# Compile a specific project
sbt "project chipyard; compile"

# Run tests for a specific project
sbt "project chipyard; test"
```

The SBT project structure is defined in `build.sbt`. Key projects: `chipyard`, `rocketchip`, `boom`, `testchipip`, `hardfloat`, `constellation`, `firechip`.

## Architecture Overview

### Hardware Hierarchy

```
TestHarness (Module, implicit clock/reset)
  └─ ChipTop (LazyRawModule, no implicit clock - all clocking explicit)
       ├─ IOBinders → create IO cells + ports per system trait
       └─ DigitalTop (the actual SoC, set via BuildSystem Field)
            ├─ ChipyardSubsystem (extends BaseSubsystem)
            │    ├─ Tiles (Rocket, BOOM, etc.)
            │    ├─ Bus topology (SBus, PBus, MBus, etc.)
            │    ├─ CLINT, PLIC, Debug
            │    └─ Memory ports
            └─ Peripheral mixins (UART, GPIO, SPI, SerialTL, etc.)
```

### Configuration System (CDE)

Chipyard uses **CDE** (Context-Dependent Environments) for parameterization. Configs are composed with `++`:

```scala
class MyConfig extends Config(
  new WithNBigCores(2) ++              // 2 Rocket cores
  new WithInclusiveCache ++            // L2 cache
  new chipyard.config.AbstractConfig   // base system defaults
)
```

- `generators/chipyard/src/main/scala/config/AbstractConfig.scala` - Default base config (defines HarnessBinders, IOBinders, memory, clocking, etc.)
- `generators/chipyard/src/main/scala/config/fragments/` - Reusable config fragments organized by domain:
  - `TileFragments.scala` - Core/tile settings
  - `SubsystemFragments.scala` - Bus/cache settings
  - `ClockingFragments.scala` - Clock domain crossings, frequencies
  - `PeripheralFragments.scala` - UART, GPIO, SPI, I2C, etc.
  - `RoCCFragments.scala` - RoCC accelerator settings
- `generators/chipyard/src/main/scala/config/RocketConfigs.scala`, `BoomConfigs.scala`, etc. - Pre-built configs

### IOBinders and HarnessBinders

This is Chipyard's key abstraction for separating chip IO from simulation harness:

1. **IOBinders** (`generators/chipyard/src/main/scala/iobinders/`) - Run inside ChipTop; for each system trait, instantiate IO cells and create ports. Configured via `IOBinders` Field.
2. **HarnessBinders** (`generators/chipyard/src/main/scala/harness/HarnessBinders.scala`) - Run inside TestHarness; match ports and connect simulation models (UART adapters, SimDRAM, SimJTAG, etc.). Configured via `HarnessBinders` Field.

Adding a new peripheral requires: (1) a mixin trait on DigitalTop, (2) an IOBinder config to expose ports, (3) a HarnessBinder config to drive them in simulation.

### Key Source Locations

- **Top-level chip:** `generators/chipyard/src/main/scala/ChipTop.scala`
- **SoC definition:** `generators/chipyard/src/main/scala/DigitalTop.scala`
- **Subsystem:** `generators/chipyard/src/main/scala/Subsystem.scala`
- **Elaboration entry point:** `generators/chipyard/src/main/scala/Generator.scala`
- **Port definitions:** `generators/chipyard/src/main/scala/iobinders/Ports.scala`
- **Clocking:** `generators/chipyard/src/main/scala/clocking/`
- **Rocket Chip core:** `generators/rocket-chip/`
- **Test infrastructure:** `generators/testchipip/` (SerialTL, SimDTM, TSI, block device, trace IO, boot control)
- **Simulation harness:** `generators/chipyard/src/main/scala/harness/`

### Simulation Backends

- **Verilator** (`sims/verilator/`) - Open-source, cycle-accurate RTL simulation
- **VCS** (`sims/vcs/`) - Synopsys commercial simulator
- **Xcelium** (`sims/xcelium/`) - Cadence commercial simulator
- **FireSim** (`sims/firesim/`) - FPGA-accelerated simulation via AWS F1

### VLSI Flow

`vlsi/` contains Hammer-based VLSI flow integration with examples for Sky130 and ASAP7 PDKs.

### Build Outputs

Simulation builds produce artifacts under `sims/<simulator>/generated-src/<MODEL_PACKAGE>.<MODEL>.<CONFIG>/`:
- `*.fir` - FIRRTL intermediate representation
- `gen-collateral/` - Generated Verilog, memory configs, file lists
- Simulator binaries: `simulator-<package>-<config>` (Verilator) or `simv-<package>-<config>` (VCS)

## Scala/Chisel Conventions

- Scala 2.13.16 with Chisel 6.7.0 (default) or Chisel 7.0.0-RC4 (experimental)
- All generators use the **Diplomacy** pattern: `LazyModule` for parameter negotiation, `LazyModuleImp` for hardware generation
- Configuration uses CDE `Field[T]` / `Config` / `Parameters` pattern, not constructor arguments
- Optional generators are discovered by checking if their git submodule is initialized (see `withInitCheck` in `build.sbt`)
