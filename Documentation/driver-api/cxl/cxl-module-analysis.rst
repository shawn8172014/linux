=============================================
CXL Driver Module Analysis and Dependencies
=============================================

:Author: Linux CXL Subsystem Analysis
:Date: 2024

Overview
========

This document provides a detailed analysis of individual CXL driver modules,
their internal structure, dependencies, and interaction patterns within the
Linux kernel CXL subsystem.

Module Directory Structure
==========================

The CXL driver framework is organized in ``/drivers/cxl/`` with the following structure::

    drivers/cxl/
    ├── core/                   # Core framework services
    │   ├── port.c             # CXL bus and port management
    │   ├── region.c           # Memory region management
    │   ├── hdm.c              # HDM decoder management
    │   ├── mbox.c             # Mailbox command interface
    │   ├── memdev.c           # Memory device core
    │   ├── regs.c             # Register access layer
    │   ├── pmem.c             # Persistent memory core
    │   ├── pci.c              # PCI integration
    │   ├── cdat.c             # CDAT table support
    │   ├── pmu.c              # Performance monitoring
    │   ├── suspend.c          # Power management
    │   ├── trace.c            # Tracing support
    │   └── core.h             # Internal definitions
    ├── cxl.h                  # Main subsystem header
    ├── cxlmem.h               # Memory device definitions
    ├── cxlpci.h               # PCI device definitions
    ├── pmu.h                  # PMU definitions
    ├── acpi.c                 # ACPI platform driver
    ├── pci.c                  # PCI device driver
    ├── mem.c                  # Memory expansion driver
    ├── pmem.c                 # Persistent memory driver
    ├── port.c                 # Port driver
    ├── security.c             # Security operations
    ├── Kconfig                # Configuration options
    └── Makefile               # Build configuration

Core Framework Modules
======================

1. Core Port Management (core/port.c)
-------------------------------------

**Purpose**: Implements the CXL bus infrastructure and device model.

**Key Responsibilities**:
  - CXL bus type registration and management
  - Device driver registration infrastructure
  - Port hierarchy management (root, switch, endpoint ports)
  - Device lifecycle management
  - Parent-child relationship establishment

**Key Data Structures**::

    struct cxl_port {
        struct device dev;              /* Device model integration */
        struct cxl_dport *parent_dport; /* Parent downstream port */
        struct list_head dports;        /* Child downstream ports */
        struct cxl_hdm *hdm;           /* HDM decoder state */
        int commit_end;                /* Committed decoder count */
        /* ... */
    };

    struct cxl_driver {
        const char *name;
        int (*probe)(struct device *dev);
        void (*remove)(struct device *dev);
        struct device_driver drv;
    };

**Key Functions**:
  - ``cxl_driver_register()`` - Register CXL drivers
  - ``devm_cxl_add_port()`` - Create and register CXL ports
  - ``cxl_bus_match()`` - Device-driver matching logic
  - ``cxl_port_probe()`` - Generic port probing

**Module Dependencies**:
  - Depends on: Linux device model, sysfs
  - Provides services to: All other CXL modules
  - Interfaces: CXL bus, device registration

2. Memory Region Management (core/region.c)
-------------------------------------------

**Purpose**: Manages CXL memory regions and interleave configuration.

**Key Responsibilities**:
  - Memory region creation and management
  - Interleave parameter configuration
  - Memory tier integration
  - Region validation and commit operations
  - QoS and performance coordination

**Key Data Structures**::

    struct cxl_region {
        struct device dev;
        struct cxl_root_decoder *cxlrd;
        enum cxl_region_mode mode;
        enum cxl_region_type type;
        struct cxl_region_params params;
        /* ... */
    };

    struct cxl_region_params {
        int nr_targets;
        resource_size_t size;
        resource_size_t res_start;
        int interleave_ways;
        int interleave_granularity;
        /* ... */
    };

**Key Functions**:
  - ``cxl_region_probe()`` - Region initialization
  - ``cxl_region_decode_commit()`` - Commit region configuration
  - ``cxl_region_setup_targets()`` - Configure interleave targets
  - ``attach_target()`` - Attach devices to regions

**Module Dependencies**:
  - Depends on: cxl_port, memory management subsystem
  - Provides services to: cxl_mem, cxl_pmem
  - Interfaces: Memory regions, NUMA integration

3. HDM Decoder Management (core/hdm.c)
--------------------------------------

**Purpose**: Manages Host Device Memory (HDM) decoders throughout the hierarchy.

**Key Responsibilities**:
  - HDM capability discovery and enumeration
  - Decoder programming and configuration
  - Address translation and routing
  - Passthrough decoder handling
  - Capacity management

**Key Data Structures**::

    struct cxl_decoder {
        struct device dev;
        int id;
        resource_size_t base;
        resource_size_t size;
        enum cxl_decoder_type type;
        enum cxl_decoder_mode mode;
        /* ... */
    };

    struct cxl_endpoint_decoder {
        struct cxl_decoder cxld;
        struct cxl_dpa_perf dpa_perf;
        enum cxl_decoder_mode mode;
        /* ... */
    };

**Key Functions**:
  - ``devm_cxl_add_hdm_decoder()`` - Add HDM decoders
  - ``cxl_hdm_decode_init()`` - Initialize decoder capabilities
  - ``cxl_dpa_alloc()`` - Allocate device physical addresses
  - ``cxl_hdm_end_decoder()`` - Configure endpoint decoders

**Module Dependencies**:
  - Depends on: cxl_port, register access layer
  - Provides services to: cxl_mem, cxl_region
  - Interfaces: Address decode, memory allocation

4. Mailbox Interface (core/mbox.c)
---------------------------------

**Purpose**: Provides standardized mailbox command interface for CXL devices.

**Key Responsibilities**:
  - Mailbox command validation and execution
  - Security and access control
  - Vendor-specific command support
  - Asynchronous operation handling
  - Command tracing and debugging

**Key Data Structures**::

    struct cxl_mem_command {
        struct cxl_command_info info;
        u32 opcode;
        u32 flags;
    };

    struct cxl_mbox_cmd {
        u16 opcode;
        void *payload_in;
        void *payload_out;
        size_t size_in;
        size_t size_out;
        u16 return_code;
        /* ... */
    };

**Key Functions**:
  - ``cxl_mbox_send_cmd()`` - Send mailbox commands
  - ``cxl_validate_cmd_from_user()`` - Validate user commands
  - ``cxl_query_cmd()`` - Query supported commands
  - ``cxl_mem_mbox_init()`` - Initialize mailbox interface

**Module Dependencies**:
  - Depends on: cxl_port, security subsystem
  - Provides services to: cxl_pci, cxl_mem, cxl_pmem
  - Interfaces: Command interface, device communication

5. Memory Device Core (core/memdev.c)
-------------------------------------

**Purpose**: Core memory device abstraction and management.

**Key Responsibilities**:
  - Memory device registration and lifecycle
  - Character device interface
  - Device attribute management
  - Memory capacity tracking
  - User-space interface provision

**Key Data Structures**::

    struct cxl_memdev {
        struct device dev;
        struct cxl_dev_state *cxlds;
        struct work_struct detach_work;
        struct cdev cdev;
        int id;
        /* ... */
    };

    struct cxl_dev_state {
        struct device *dev;
        struct cxl_mailbox *mbox;
        resource_size_t component_reg_phys;
        void __iomem *regs;
        /* ... */
    };

**Key Functions**:
  - ``devm_cxl_add_memdev()`` - Create memory devices
  - ``cxl_memdev_probe()`` - Memory device initialization
  - ``cxl_memdev_ioctl()`` - User-space interface
  - ``cxl_memdev_shutdown()`` - Device shutdown handling

**Module Dependencies**:
  - Depends on: cxl_port, character device interface
  - Provides services to: cxl_mem, user-space applications
  - Interfaces: /dev/cxl/memX devices, sysfs attributes

Driver Modules
==============

1. ACPI Platform Driver (acpi.c)
--------------------------------

**Purpose**: Integrates with ACPI firmware for platform discovery and configuration.

**Key Responsibilities**:
  - CEDT (CXL Early Discovery Table) parsing
  - Host bridge discovery and initialization
  - CFMWS (CXL Fixed Memory Window Structure) processing
  - Platform resource allocation
  - NUMA topology integration

**Key Functions**:
  - ``cxl_acpi_probe()`` - Platform initialization
  - ``cxl_parse_cedt()`` - Parse CEDT table
  - ``cxl_parse_cfmws()`` - Process memory windows
  - ``cxl_acpi_add_cfmws_decoders()`` - Create root decoders

**ACPI Integration Points**:
  - CEDT table parsing for early discovery
  - CHBS (CXL Host Bridge Structure) processing
  - CXIMS (CXL XOR Interleave Math) support
  - QoS information extraction

**Module Dependencies**:
  - Depends on: ACPI subsystem, cxl_core
  - Provides: Platform discovery, root port creation
  - Kernel Config: ``CONFIG_CXL_ACPI=y/m``

2. PCI Device Driver (pci.c)
----------------------------

**Purpose**: Manages PCI-specific aspects of CXL devices.

**Key Responsibilities**:
  - PCI device enumeration and initialization
  - Component register mapping
  - Mailbox interface setup
  - Interrupt handling (MSI/MSI-X)
  - Device-specific configuration

**Key Functions**:
  - ``cxl_pci_probe()`` - PCI device initialization
  - ``cxl_pci_setup_mailbox()`` - Setup mailbox interface
  - ``cxl_map_component_regs()`` - Map component registers
  - ``cxl_request_irq()`` - Setup interrupt handling

**PCI Integration**:
  - Standard PCI enumeration support
  - CXL-specific capability discovery
  - Memory-mapped I/O setup
  - Power management integration

**Module Dependencies**:
  - Depends on: PCI subsystem, cxl_core
  - Provides: Device initialization, register access
  - Kernel Config: ``CONFIG_CXL_PCI=y/m``

3. Memory Expansion Driver (mem.c)
----------------------------------

**Purpose**: Implements memory expansion functionality for CXL devices.

**Key Responsibilities**:
  - Endpoint port creation and management
  - Device Physical Address (DPA) management
  - Memory topology discovery
  - HDM decoder configuration
  - Memory region enablement

**Key Functions**:
  - ``cxl_mem_probe()`` - Memory driver initialization
  - ``devm_cxl_add_endpoint()`` - Create endpoint ports
  - ``cxl_dpa_set_mode()`` - Configure DPA mode
  - ``cxl_mem_create_range_info()`` - Setup memory ranges

**Memory Management Integration**:
  - System RAM integration
  - Memory tier support
  - NUMA node assignment
  - Performance characteristic reporting

**Module Dependencies**:
  - Depends on: cxl_pci, cxl_core, memory management
  - Provides: Memory expansion, endpoint management
  - Kernel Config: ``CONFIG_CXL_MEM=y/m``

4. Persistent Memory Driver (pmem.c + security.c)
-------------------------------------------------

**Purpose**: Bridges CXL persistent memory to the LIBNVDIMM subsystem.

**Key Responsibilities**:
  - LIBNVDIMM integration and bridge creation
  - NVDIMM namespace management
  - Label storage area support
  - Security operations (authentication, secure erase)
  - Persistent memory region management

**Key Functions**:
  - ``cxl_pmem_probe()`` - PMEM driver initialization  
  - ``cxl_nvdimm_bridge_probe()`` - Create NVDIMM bridge
  - ``cxl_pmem_ctl()`` - NVDIMM control interface
  - ``cxl_security_freeze()`` - Security state management

**LIBNVDIMM Integration**:
  - NVDIMM bus registration
  - Namespace creation and management
  - Label storage area (LSA) support
  - Security operation forwarding

**Module Dependencies**:
  - Depends on: LIBNVDIMM, cxl_core, cxl_mem
  - Provides: Persistent memory support, NVDIMM bridge
  - Kernel Config: ``CONFIG_CXL_PMEM=y/m``

5. Port Driver (port.c)
-----------------------

**Purpose**: Manages port-specific operations and lifecycle.

**Key Responsibilities**:
  - Port device binding and management
  - Switch port operations
  - Endpoint port operations
  - Port-specific attribute management
  - Decoder chain management

**Key Functions**:
  - ``cxl_port_probe()`` - Port driver initialization
  - ``cxl_port_setup_targets()`` - Configure port targets
  - ``cxl_port_attach_region()`` - Attach regions to ports
  - ``cxl_port_detach_region()`` - Detach regions from ports

**Port Type Management**:
  - Root port operations
  - Switch upstream/downstream ports
  - Endpoint port management
  - Port hierarchy validation

**Module Dependencies**:
  - Depends on: cxl_core
  - Provides: Port-specific operations
  - Kernel Config: ``CONFIG_CXL_PORT=y/m`` (auto-selected)

Module Dependency Matrix
========================

The following matrix shows the dependency relationships between CXL modules:

+----------------+----------+----------+----------+----------+----------+
| Module         | cxl_core | cxl_acpi | cxl_pci  | cxl_mem  | cxl_pmem |
+================+==========+==========+==========+==========+==========+
| **cxl_core**   |    -     |    -     |    -     |    -     |    -     |
+----------------+----------+----------+----------+----------+----------+
| **cxl_acpi**   |    ✓     |    -     |    -     |    -     |    -     |
+----------------+----------+----------+----------+----------+----------+
| **cxl_pci**    |    ✓     |    -     |    -     |    -     |    -     |
+----------------+----------+----------+----------+----------+----------+
| **cxl_mem**    |    ✓     |    -     |    ✓     |    -     |    -     |
+----------------+----------+----------+----------+----------+----------+
| **cxl_pmem**   |    ✓     |    -     |    -     |    ✓     |    -     |
+----------------+----------+----------+----------+----------+----------+
| **cxl_port**   |    ✓     |    -     |    -     |    -     |    -     |
+----------------+----------+----------+----------+----------+----------+

Legend:
  - ✓ = Direct dependency
  - - = No direct dependency

Initialization Order
====================

The CXL modules are initialized in the following order to satisfy dependencies:

1. **cxl_core** (built-in or loaded first)
   - Registers CXL bus type
   - Initializes core services
   - Sets up device model infrastructure

2. **cxl_acpi** (if present and ACPI available)
   - Parses CEDT table
   - Creates root ports
   - Sets up platform resources

3. **cxl_pci** (when CXL PCI devices are discovered)
   - Enumerates PCI devices
   - Maps registers
   - Creates memory devices

4. **cxl_mem** (depends on cxl_pci)
   - Creates endpoint ports
   - Configures memory regions
   - Enables memory expansion

5. **cxl_pmem** (depends on cxl_mem and LIBNVDIMM)
   - Creates NVDIMM bridge
   - Sets up persistent memory support

6. **cxl_port** (auto-loaded as needed)
   - Handles port-specific operations

Communication Patterns
======================

Inter-module Communication
--------------------------

1. **Service Registration Pattern**::
    
    cxl_acpi → cxl_core: register_platform_resources()
    cxl_pci → cxl_core: register_memory_device()
    cxl_mem → cxl_core: register_endpoint_port()

2. **Callback Pattern**::
    
    cxl_core → cxl_port: probe/remove callbacks
    cxl_core → cxl_mem: region_attach/detach callbacks

3. **Shared Data Structures**::
    
    All modules access core data structures through cxl_core APIs
    - struct cxl_port for port management
    - struct cxl_region for region operations
    - struct cxl_memdev for device access

Configuration Dependencies
==========================

Kernel Configuration Options
----------------------------

**Essential Configuration**::

    CONFIG_CXL_BUS=y/m              # Core CXL support
    CONFIG_PCI=y                    # PCI subsystem (required)
    CONFIG_ACPI=y                   # ACPI support (for platform)
    CONFIG_SPARSEMEM=y              # Sparse memory model

**Module-Specific Configuration**::

    CONFIG_CXL_PCI=y/m              # PCI device support
    CONFIG_CXL_MEM=y/m              # Memory expansion
    CONFIG_CXL_ACPI=y/m             # ACPI platform support
    CONFIG_CXL_PMEM=y/m             # Persistent memory
    CONFIG_CXL_REGION=y             # Region support

**Optional Features**::

    CONFIG_CXL_MEM_RAW_COMMANDS=y   # Raw command interface
    CONFIG_CXL_SUSPEND=y            # Suspend/resume support
    CONFIG_CXL_REGION_INVALIDATION_TEST=y  # Testing support

Build Dependencies
------------------

**Makefile Structure**::

    obj-y += core/                   # Core always built
    obj-$(CONFIG_CXL_PCI) += cxl_pci.o
    obj-$(CONFIG_CXL_MEM) += cxl_mem.o  
    obj-$(CONFIG_CXL_ACPI) += cxl_acpi.o
    obj-$(CONFIG_CXL_PMEM) += cxl_pmem.o
    obj-$(CONFIG_CXL_PORT) += cxl_port.o

Error Handling and Recovery
===========================

Module Error Handling Patterns
------------------------------

1. **Graceful Degradation**:
   - If cxl_acpi fails, PCI-only operation continues
   - If cxl_pmem fails, volatile memory operation continues
   - Individual device failures don't affect others

2. **Resource Cleanup**:
   - All modules use devm_ functions for automatic cleanup
   - Module removal triggers cascading cleanup
   - Reference counting prevents use-after-free

3. **Error Propagation**:
   - Critical errors propagate up the hierarchy
   - Non-critical errors are logged and contained
   - Recovery mechanisms attempt automatic retry

Future Evolution
================

Module Extensibility
--------------------

The CXL driver framework is designed for future expansion:

1. **New CXL Specification Features**:
   - Additional modules can be added for new capabilities
   - Existing modules can be extended with new interfaces
   - Core framework provides stable API

2. **Platform Integration**:
   - Additional platform drivers (beyond ACPI)
   - Alternative firmware integration paths
   - Enhanced QoS and performance features

3. **Memory Technology Evolution**:
   - Support for new memory types
   - Enhanced persistent memory features
   - Improved performance monitoring

Conclusion
==========

The CXL driver module architecture provides:

- **Clear Separation of Concerns**: Each module has well-defined responsibilities
- **Layered Dependencies**: Core services support specialized functionality
- **Flexible Configuration**: Modules can be enabled independently based on needs
- **Robust Error Handling**: Graceful degradation and recovery mechanisms
- **Future Extensibility**: Framework supports evolution and new features

This modular design enables efficient development, testing, and maintenance while
providing the flexibility needed for diverse CXL deployment scenarios.