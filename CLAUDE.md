# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# MOON-SIM: Modular Object-Oriented Network Simulator

**MOON-SIM** is a high-performance SystemC-based SoC architecture simulation system designed for storage systems and network-on-chip modeling.

## Project Philosophy

- **M**odular: Independent, reusable components with single responsibilities
- **O**bject-**O**riented: Template-based design with polymorphic packet systems
- **N**etwork: Event-driven communication with PCIe-style bidirectional architecture
- **SIM**ulator: High-performance SystemC simulation achieving 25M+ TPS

## Build and Development Commands

### Environment Setup (Required)
```bash
# Set SYSTEMC_HOME environment variable
export SYSTEMC_HOME=/home/stone/project/systemc/systemc-install

# Or for different installations:
export SYSTEMC_HOME=/usr/local/systemc        # System installation
export SYSTEMC_HOME=/opt/systemc              # Custom location

# Add to ~/.bashrc for permanent setting:
echo 'export SYSTEMC_HOME=/home/stone/project/systemc/systemc-install' >> ~/.bashrc
source ~/.bashrc
```

### Build the simulation (C++11 with custom SystemC)
```bash
make clean && make
```

### Run the simulation
```bash
./sim                           # Use default config/ directory
./sim config/sweeps/my_test/    # Use specific configuration
```

### Run parameter sweeps
```bash
python3 run_sweep.py config/sweeps/small_test.json
python3 run_sweep.py config/sweeps/write_ratio_sweep.json
```

### SystemC Environment
- **Environment Variable**: `SYSTEMC_HOME` must be set (Makefile enforces this)
- **Custom C++11 SystemC**: Built from SystemC 2.3.3 source with `-std=c++11`
- **Default Path**: `/home/stone/project/systemc/systemc-install`
- **Corporate Compatible**: Works in C++11-only environments
- **Flexible Installation**: Supports any SystemC installation path via environment variable

## MOON-SIM Architecture

MOON-SIM implements a high-performance SystemC-based SoC architecture simulation system with a PCIe-style bidirectional design:

**HostSystem (TrafficGenerator + IndexAllocator) ⟷ Memory**
- **Downstream**: HostSystem → DownstreamDelay → Memory  
- **Upstream**: Memory → UpstreamDelay → HostSystem (for index release)

### Core Components

1. **HostSystem** (`src/host_system/host_system.cpp`)
   - Encapsulates TrafficGenerator and IndexAllocator as a unified module
   - Separate JSON configuration file (`config/host_system_config.json`)
   - Internal FIFO communication between TrafficGenerator and IndexAllocator
   - PCIe-style bidirectional interface (downstream out, upstream release_in)

2. **TrafficGenerator** (`src/base/traffic_generator.cpp`)
   - Generates READ/WRITE packets with percentage-based controls:
     - **Locality**: 0-100% (0=full random, 100=full sequential)
     - **Write Ratio**: 0-100% (0=all reads, 100=all writes)
   - Configurable address ranges (start_address, end_address, address_increment)
   - Separate JSON configuration file (`config/traffic_generator_config.json`)
   - Runtime debug control via JSON configuration

3. **IndexAllocator** (`include/base/index_allocator.h` - Template)
   - **Resource-limited allocation**: Configurable max_index pool with blocking when exhausted
   - **Automatic recycling**: Index deallocation when Memory processing completes
   - **Bidirectional operation**: Allocation on request path, deallocation on release path
   - Multiple allocation policies: Sequential, Round-Robin, Random, Pool-based
   - Template-based packet-type independence with custom index setter functions

4. **DelayLine** (`include/base/delay_line.h` - Template)
   - **PCIe-style bidirectional**: Separate downstream and upstream DelayLines
   - Header-only template component for any packet type
   - Configurable fixed delay simulation (nanosecond precision)
   - Runtime debug logging control

5. **Memory** (`include/base/memory.h` - Template)
   - Template-based scalable memory (256 to 65536+ entries)
   - **Release path**: Automatic packet return for index deallocation
   - Configurable data types and memory sizes via template parameters
   - Bounds checking with comprehensive error handling

### Packet System

- **BasePacket** (`include/packet/base_packet.h`): Abstract base class with virtual methods and set_attribute support
- **GenericPacket** (`include/packet/generic_packet.h`): Concrete implementation with index field and dynamic attribute setting
- Polymorphic design with virtual methods for extensibility and type safety

### Communication

- **PCIe-style bidirectional**: Separate downstream and upstream paths
- Components communicate via `sc_fifo<std::shared_ptr<BasePacket>>` 
- Smart pointers ensure automatic memory management and performance
- **Index lifecycle management**: Allocation → Processing → Release → Recycling

### Configuration System

- **Modular JSON Configuration**: Separate config files for better organization
  - `config/simulation_config.json`: Main simulation and DelayLine settings
  - `config/traffic_generator_config.json`: TrafficGenerator-specific settings
  - `config/host_system_config.json`: HostSystem and IndexAllocator settings
- **Per-component debug control**: Enable/disable logging for performance
- **Resource management**: Configurable index pool sizes and allocation policies
- **Performance optimization**: I/O bottleneck elimination achieving 25M+ tps

### Performance Features

- **Ultra-High Throughput**: 25,000,000+ transactions/second (PCIe-style architecture)
- **Index Resource Management**: Blocking allocation prevents resource exhaustion
- **Template Design**: Header-only templates for optimal performance
- **Smart Logging**: Runtime enable/disable to eliminate I/O bottlenecks
- **Memory Efficiency**: Shared pointer management with automatic index recycling

### Utilities

- **common_utils.h**: Packet logging and dynamic attribute access functions
- **json_config.h**: Simple JSON parser for runtime configuration
- **error_handling.h**: Centralized error codes and reporting
- Consistent timestamped logging with performance control

### SystemC Dependencies

- Requires SystemC 2.3.4+ installation at `/usr/local/systemc`
- Uses SystemC processes, modules, and timing mechanisms
- Simulation logs automatically timestamped and saved to `log/` directory
- Template-based design follows SystemC best practices

### File Organization

```
include/
├── base/           # Template components (delay lines, memory, traffic gen, index allocator)
├── common/         # Utilities (JSON config, error handling, common utils)
├── packet/         # Packet definitions and base classes
└── host_system/    # HostSystem encapsulation module

src/
├── base/           # Non-template implementations (traffic_generator.cpp only)
├── host_system/    # HostSystem implementation (host_system.cpp)
└── main.cpp        # Simulation entry point with PCIe-style architecture

config/
├── simulation_config.json         # Main simulation settings
├── traffic_generator_config.json  # TrafficGenerator-specific settings
└── host_system_config.json       # HostSystem and IndexAllocator settings

log/                # Timestamped simulation output logs
```

### JSON Configuration Examples

#### `config/traffic_generator_config.json`
```json
{
  "traffic_generator": {
    "num_transactions": 25,
    "interval_ns": 10,
    "locality_percentage": 100,
    "write_percentage": 100,
    "databyte_value": 64,
    "debug_enable": false,
    "start_address": 0,
    "end_address": 65535,
    "address_increment": 64
  }
}
```

#### `config/host_system_config.json`
```json
{
  "host_system": {
    "index_allocator": {
      "policy": "SEQUENTIAL",
      "max_index": 10,
      "enable_reuse": true,
      "debug_enable": true
    }
  }
}
```

#### `config/simulation_config.json`
```json
{
  "simulation": {
    "delay_lines": {
      "delay_ns": 10,
      "debug_enable": false
    },
    "memory": {
      "size": 256,
      "debug_enable": false
    }
  },
  "logging": {
    "enable_file_logging": true,
    "performance_metrics": true
  }
}
```

### Parameter Sweep Configuration Examples

#### `config/sweeps/small_test.json`
```json
{
  "sweep_name": "small_write_test",
  "description": "Small test with only 3 points",
  "base_config": "config",
  "parameter": "write_percentage",
  "config_file": "traffic_generator_config.json",
  "start": 0,
  "end": 20,
  "step": 10,
  "expected_points": 3,
  "version": "1.0"
}
```

#### `config/sweeps/write_ratio_sweep.json`
```json
{
  "sweep_name": "write_percentage_sweep",
  "description": "Sweep write_percentage from 0% to 100% in 10% steps",
  "base_config": "config",
  "parameter": "write_percentage",
  "config_file": "traffic_generator_config.json",
  "start": 0,
  "end": 100,
  "step": 10,
  "expected_points": 11,
  "version": "1.0"
}
```

### Performance Benchmarks (C++11 Environment)

- **PCIe-Style Architecture (Debug Off)**: 25,000,000+ transactions/second
- **Index Resource Management (100 pool)**: 10,000,000+ transactions/second
- **100% Write Ratio**: 25,000,000+ transactions/second
- **Mixed Read/Write (50%)**: 15,000,000+ transactions/second
- **Debug Enabled**: ~2,000-3,000 transactions/second
- **Parameter Sweeps**: TC001-TC003 format with performance metrics

### C++11 Compatibility Notes

- **No std::make_unique**: Uses `std::unique_ptr<T>(new T(...))` for C++11 compatibility
- **Custom SystemC Build**: SystemC 2.3.3 compiled with `-std=c++11` standard
- **Template Components**: All performance-critical components are header-only templates
- **Corporate Environment Ready**: Tested in C++11-only environments

### Design Principles: Modular Architecture

Always follow these modular design principles when developing or modifying the SystemC simulation:

#### 0. **Event-Driven Communication (CRITICAL)**
- **NEVER use polling loops**: Avoid `while(condition) { wait(1, SC_NS); }` patterns
- **Use sc_event for process communication**: All inter-process communication must be event-driven
- **Notify events when data is available**: Use `event.notify()` to wake up waiting processes
- **Wait on events, not time**: Use `wait(event)` instead of `wait(time)` for synchronization
- **Performance Impact**: Polling loops can reduce simulation performance by 100x or more

```cpp
// ❌ NEVER DO THIS - Inefficient polling
while (queue.empty()) {
    wait(1, SC_NS);  // Kills performance!
}

// ✅ ALWAYS DO THIS - Event-driven
sc_event data_ready;
// Producer: data_ready.notify();
// Consumer: wait(data_ready);
```

#### 1. **Avoid Duplication**
- Never create redundant components that replicate existing functionality
- Reuse existing modules (e.g., HostSystem's embedded profiler) instead of creating separate ones
- If similar functionality exists, extend or configure the existing component rather than duplicating

#### 2. **Single Responsibility Principle**
- Each module should have one clear, well-defined responsibility
- HostSystem handles traffic generation and profiling
- DelayLines handle timing simulation
- Memory components handle storage simulation
- Avoid mixing concerns within a single module

#### 3. **Leverage Built-in Capabilities**
- Use HostSystem's embedded ProfilerBW and ProfilerLatency instead of external monitors
- Utilize existing template components before creating new ones
- Check existing modules for required functionality before implementing new features

#### 4. **Prefer Composition Over Inheritance**
- Use template-based composition for performance-critical paths
- Encapsulate related components (e.g., TrafficGenerator + IndexAllocator → HostSystem)
- Maintain clear interfaces between modules using SystemC FIFOs

#### 5. **Configuration-Driven Design**
- Make components configurable through JSON rather than hardcoded values
- Separate configuration concerns (separate JSON files for different components)
- Enable/disable features through configuration rather than conditional compilation

#### 6. **Performance-First Architecture**
- Minimize intermediate components in critical paths
- Use direct connections where possible (HostSystem → PCIe → Target)
- Prefer header-only templates for performance-critical components
- Avoid unnecessary FIFO buffers and intermediate modules

#### 7. **Template-Based Reusability**
- Design components as templates to work with different packet types
- Use template parameters for sizing and configuration
- Maintain type safety while enabling flexibility

#### Examples of Good Modular Design:
```cpp
// ✅ Good: Reuse existing HostSystem profiler
host_system.force_profiler_report();

// ❌ Bad: Create redundant latency monitor
SSDLatencyMonitor custom_monitor("monitor", profiler);

// ✅ Good: Direct, efficient connections
HostSystem → PCIe → SSD

// ❌ Bad: Unnecessary intermediate components
HostSystem → CustomMonitor → PCIe → SSD
```

#### Code Review Checklist:
- [ ] Does this duplicate existing functionality?
- [ ] Can existing components be reused or extended instead?
- [ ] Does each module have a single, clear responsibility?
- [ ] Are connections as direct as possible for performance?
- [ ] Is the design configurable rather than hardcoded?
- [ ] Are templates used appropriately for reusability?