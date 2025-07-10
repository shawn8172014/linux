========================================
CXL Driver Framework Architecture Diagrams
========================================

This document provides detailed diagrams illustrating the CXL driver framework
architecture, module interactions, and system integration.

1. Overall Framework Architecture
=================================

```
                    Linux CXL Driver Framework Architecture
                           
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                          USER SPACE                                     │
    ├─────────────────────────────────────────────────────────────────────────┤
    │    sysfs        │    character     │    LIBNVDIMM     │    Memory       │
    │  interfaces     │    devices       │    interface     │   Management    │
    └─────┬───────────┴──────┬───────────┴──────┬───────────┴──────┬──────────┘
          │                  │                  │                  │
    ┌─────┴─────────┬────────┴─────────┬────────┴─────────┬────────┴──────────┐
    │               │                  │                  │                   │
    │  ┌─────────┐  │  ┌─────────────┐ │  ┌─────────────┐ │  ┌─────────────┐  │
    │  │ sysfs   │  │  │ cxl_mem     │ │  │ cxl_pmem    │ │  │ mm subsys   │  │
    │  │ core    │  │  │ char dev    │ │  │ nvdimm      │ │  │ integration │  │
    │  └─────────┘  │  └─────────────┘ │  │ bridge      │ │  └─────────────┘  │
    │               │                  │  └─────────────┘ │                   │
    └───────────────┴──────────────────┴──────────────────┴───────────────────┘
                                       │
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        CXL DRIVER MODULES                               │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │ cxl_acpi    │  │ cxl_pci     │  │ cxl_mem     │  │ cxl_pmem    │    │
    │  │             │  │             │  │             │  │             │    │
    │  │ - CEDT      │  │ - PCI enum  │  │ - Memory    │  │ - NVDIMM    │    │
    │  │   parsing   │  │ - Mailbox   │  │   expansion │  │   bridge    │    │
    │  │ - Platform  │  │ - Register  │  │ - Endpoint  │  │ - Security  │    │
    │  │   discovery │  │   mapping   │  │   creation  │  │ - Labels    │    │
    │  │ - Host      │  │ - IRQ       │  │ - DPA mgmt  │  │             │    │
    │  │   bridges   │  │   handling  │  │             │  │             │    │
    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                    cxl_port                                     │   │
    │  │  - Port management and lifecycle                                │   │
    │  │  - Switch and endpoint port operations                         │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                          CXL CORE FRAMEWORK                             │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │    port.c   │  │   region.c  │  │    hdm.c    │  │   mbox.c    │    │
    │  │             │  │             │  │             │  │             │    │
    │  │ - CXL bus   │  │ - Memory    │  │ - HDM       │  │ - Mailbox   │    │
    │  │ - Driver    │  │   regions   │  │   decoders  │  │   commands  │    │
    │  │   model     │  │ - Interleave│  │ - Address   │  │ - Security  │    │
    │  │ - Port      │  │   config    │  │   decode    │  │ - Command   │    │
    │  │   hierarchy │  │ - Memory    │  │ - Capacity  │  │   validation│    │
    │  │             │  │   tiers     │  │   mgmt      │  │             │    │
    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │  memdev.c   │  │   regs.c    │  │   pmem.c    │  │   cdat.c    │    │
    │  │             │  │             │  │             │  │             │    │
    │  │ - Memory    │  │ - Register  │  │ - LIBNVDIMM │  │ - QoS info  │    │
    │  │   devices   │  │   access    │  │   support   │  │ - CDAT      │    │
    │  │ - Device    │  │ - MMIO      │  │ - Namespace │  │   parsing   │    │
    │  │   attributes│  │   mapping   │  │   mgmt      │  │ - Memory    │    │
    │  │ - Char dev  │  │ - RCRB      │  │             │  │   attributes│    │
    │  │   interface │  │   support   │  │             │  │             │    │
    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐                                      │
    │  │   pmu.c     │  │ suspend.c   │                                      │
    │  │             │  │             │                                      │
    │  │ - Perf      │  │ - Power     │                                      │
    │  │   monitoring│  │   mgmt      │                                      │
    │  │ - Counters  │  │ - Suspend/  │                                      │
    │  │ - Events    │  │   resume    │                                      │
    │  └─────────────┘  └─────────────┘                                      │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        HARDWARE ABSTRACTION                             │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │ PCI Subsys  │  │ ACPI Subsys │  │ Memory      │  │ Interrupt   │    │
    │  │             │  │             │  │ Subsystem   │  │ Subsystem   │    │
    │  │ - Device    │  │ - CEDT      │  │ - Physical  │  │ - MSI/MSI-X │    │
    │  │   enumeration│  │   parsing   │  │   memory    │  │ - IRQ       │    │
    │  │ - Config    │  │ - Namespace │  │ - DMA       │  │   routing   │    │
    │  │   space     │  │   integration│  │ - NUMA      │  │             │    │
    │  │ - BARs      │  │ - NUMA info │  │             │  │             │    │
    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                            HARDWARE LAYER                               │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │ Host Bridge │  │ CXL Switch  │  │ CXL Endpoint│  │ Memory      │    │
    │  │             │  │             │  │             │  │ Devices     │    │
    │  │ - Root      │  │ - Upstream  │  │ - Type 3    │  │             │    │
    │  │   complex   │  │   port      │  │   device    │  │ - DDR/HBM   │    │
    │  │ - PCIe      │  │ - Downstream│  │ - Mailbox   │  │ - Persistent│    │
    │  │   root port │  │   ports     │  │   interface │  │   memory    │    │
    │  │ - CEDT      │  │ - Fabric    │  │ - HDM       │  │ - NVM       │    │
    │  │   entries   │  │   mgmt      │  │   decoders  │  │             │    │
    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
    └─────────────────────────────────────────────────────────────────────────┘
```

2. Module Interaction Diagram
=============================

```
                       CXL Driver Module Interactions

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         Initialization Flow                             │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
    ┌─────────────────┐                │                ┌─────────────────┐
    │ UEFI/ACPI       │                │                │ PCI Subsystem   │
    │ - CEDT Table    │──────────┐     │     ┌──────────│ - Device        │
    │ - Platform      │          │     │     │          │   Enumeration   │
    │   Configuration │          │     │     │          │ - CXL Class     │
    └─────────────────┘          │     │     │          │   Code Match    │
                                 │     │     │          └─────────────────┘
                                 ▼     │     ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                            cxl_core                                     │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                     Core Services                               │   │
    │  │                                                                 │   │
    │  │  Bus Registration  ──►  Device Model  ──►  Driver Registration │   │
    │  │        │                     │                      │          │   │
    │  │        ▼                     ▼                      ▼          │   │
    │  │  Port Management  ──►  Region Management  ──►  HDM Management  │   │
    │  │        │                     │                      │          │   │
    │  │        ▼                     ▼                      ▼          │   │
    │  │  Mailbox Core    ──►  Memory Management  ──►  Register I/O    │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    └─────────────────────────────────────────────────────────────────────────┘
              │                       │                       │
              ▼                       ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
    │   cxl_acpi      │     │    cxl_pci      │     │    cxl_mem      │
    │                 │     │                 │     │                 │
    │ ┌─────────────┐ │     │ ┌─────────────┐ │     │ ┌─────────────┐ │
    │ │ CEDT Parse  │ │     │ │ PCI Driver  │ │     │ │ Memory      │ │
    │ │     │       │ │     │ │     │       │ │     │ │ Driver      │ │
    │ │     ▼       │ │     │ │     ▼       │ │     │ │     │       │ │
    │ │ Create Root │ │     │ │ Register    │ │     │ │     ▼       │ │
    │ │ Ports       │ │     │ │ Mapping     │ │     │ │ Create      │ │
    │ │     │       │ │     │ │     │       │ │     │ │ Endpoints   │ │
    │ │     ▼       │ │     │ │     ▼       │ │     │ │     │       │ │
    │ │ Setup       │ │     │ │ Mailbox     │ │     │ │     ▼       │ │
    │ │ CFMWS       │ │     │ │ Init        │ │     │ │ DPA         │ │
    │ │     │       │ │     │ │     │       │ │     │ │ Management  │ │
    │ │     ▼       │ │     │ │     ▼       │ │     │ │     │       │ │
    │ │ Root        │ │     │ │ Create      │ │     │ │     ▼       │ │
    │ │ Decoder     │ │     │ │ Memdev      │ │     │ │ HDM Config  │ │
    │ │ Config      │ │     │ │             │ │     │ │             │ │
    │ └─────────────┘ │     │ └─────────────┘ │     │ └─────────────┘ │
    └─────────────────┘     └─────────────────┘     └─────────────────┘
              │                       │                       │
              └───────────────────────┼───────────────────────┘
                                     │
                                     ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        cxl_pmem & cxl_port                              │
    │                                                                         │
    │  ┌─────────────────┐                      ┌─────────────────┐           │
    │  │   cxl_pmem      │                      │   cxl_port      │           │
    │  │                 │                      │                 │           │
    │  │ ┌─────────────┐ │                      │ ┌─────────────┐ │           │
    │  │ │ NVDIMM      │ │                      │ │ Port        │ │           │
    │  │ │ Bridge      │ │                      │ │ Operations  │ │           │
    │  │ │     │       │ │                      │ │     │       │ │           │
    │  │ │     ▼       │ │                      │ │     ▼       │ │           │
    │  │ │ Namespace   │ │                      │ │ Switch      │ │           │
    │  │ │ Management  │ │                      │ │ Management  │ │           │
    │  │ │     │       │ │                      │ │     │       │ │           │
    │  │ │     ▼       │ │                      │ │     ▼       │ │           │
    │  │ │ Security    │ │                      │ │ Endpoint    │ │           │
    │  │ │ Operations  │ │                      │ │ Operations  │ │           │
    │  │ └─────────────┘ │                      │ └─────────────┘ │           │
    │  └─────────────────┘                      └─────────────────┘           │
    └─────────────────────────────────────────────────────────────────────────┘
```

3. UEFI/Firmware Interaction UML Diagram
========================================

```
                    CXL Driver - UEFI/Firmware Interaction

    ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
    │ UEFI/BIOS    │              │ ACPI Tables  │              │ cxl_acpi     │
    │              │              │              │              │ Driver       │
    └──────┬───────┘              └──────┬───────┘              └──────┬───────┘
           │                             │                             │
           │ System Boot                 │                             │
           ├─────────────────────────────┼────────────────────────────►│
           │                             │                             │
           │ Create ACPI Tables          │                             │
           ├────────────────────────────►│                             │
           │                             │                             │
           │  ┌─────────────────────────┐│                             │
           │  │ CEDT Creation           ││                             │
           │  │ - CHBS (Host Bridge)    ││                             │
           │  │ - CFMWS (Memory Window) ││                             │
           │  │ - CXIMS (Interleave)    ││                             │
           │  └─────────────────────────┘│                             │
           │                             │                             │
           │ OS Handoff                  │                             │
           └─────────────────────────────┼────────────────────────────►│
                                         │                             │
                                         │ Parse CEDT                  │
                                         │◄────────────────────────────┤
                                         │                             │
                                         │ Return CEDT Data            │
                                         ├────────────────────────────►│
                                         │                             │
                                         │                             │
    ┌──────────────┐                     │              ┌──────────────┤
    │ PCI Config   │                     │              │ Platform     │
    │ Space        │                     │              │ Discovery    │
    └──────┬───────┘                     │              └──────┬───────┘
           │                             │                     │
           │ CXL Capability Discovery    │                     │
           │◄────────────────────────────┼─────────────────────┤
           │                             │                     │
           │ Return Capability Info      │                     │
           ├─────────────────────────────┼────────────────────►│
           │                             │                     │
           │                             │                     │
    ┌──────┴───────┐              ┌──────┴───────┐              ┌──────┴───────┐
    │ Component    │              │ Mailbox      │              │ Port         │
    │ Registers    │              │ Interface    │              │ Hierarchy    │
    │              │              │              │              │ Creation     │
    └──────┬───────┘              └──────┬───────┘              └──────┬───────┘
           │                             │                             │
           │ HDM Decoder Discovery       │                             │
           │◄────────────────────────────┼─────────────────────────────┤
           │                             │                             │
           │ Return Decoder Info         │                             │
           ├─────────────────────────────┼────────────────────────────►│
           │                             │                             │
           │                             │ Mailbox Commands            │
           │                             │◄────────────────────────────┤
           │                             │                             │
           │                             │ Device Information          │
           │                             ├────────────────────────────►│
           │                             │                             │
           │                             │                             │
           │ Memory Window Programming   │                             │
           │◄────────────────────────────┼─────────────────────────────┤
           │                             │                             │
           │ Configuration Complete      │                             │
           ├─────────────────────────────┼────────────────────────────►│
           │                             │                             │

    Sequence of Operations:
    1. UEFI/BIOS creates ACPI tables during boot
    2. CEDT table contains CXL platform configuration
    3. cxl_acpi driver parses CEDT during initialization
    4. Platform discovery creates root ports and decoders
    5. Component register discovery for capabilities
    6. Mailbox interface initialization for device communication
    7. Memory window and decoder programming
```

4. Hardware Interaction Diagram
===============================

```
                        CXL Driver - Hardware Interaction

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                          Software Stack                                 │
    └─────────────────────┬───────────────────────┬───────────────────────────┘
                          │                       │
    ┌─────────────────────▼───────────────────────▼───────────────────────────┐
    │                     CXL Driver Framework                                │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │ cxl_acpi    │  │ cxl_pci     │  │ cxl_mem     │  │ cxl_pmem    │    │
    │  │ (Platform)  │  │ (Device)    │  │ (Memory)    │  │ (Persistent)│    │
    │  └─────┬───────┘  └─────┬───────┘  └─────┬───────┘  └─────┬───────┘    │
    │        │                │                │                │            │
    │        │                │                │                │            │
    │  ┌─────┴─────┐    ┌─────┴─────┐    ┌─────┴─────┐    ┌─────┴─────┐      │
    │  │   ACPI    │    │   PCI     │    │  Memory   │    │ LIBNVDIMM │      │
    │  │Interface  │    │Interface  │    │Management │    │ Interface │      │
    │  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘      │
    └────────┼────────────────┼────────────────┼────────────────┼────────────┘
             │                │                │                │
    ┌────────┼────────────────┼────────────────┼────────────────┼────────────┐
    │        │                │                │                │            │
    │        ▼                ▼                ▼                ▼            │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
    │  │    ACPI     │  │     PCI     │  │   Memory    │  │ Persistent  │    │
    │  │ Namespace   │  │ Config      │  │ Controller  │  │ Memory      │    │
    │  │             │  │ Space       │  │             │  │ Controller  │    │
    │  │ - CEDT      │  │             │  │ - DMA       │  │             │    │
    │  │ - CHBS      │  │ - Device    │  │ - NUMA      │  │ - NVDIMMs   │    │
    │  │ - CFMWS     │  │   Control   │  │ - Memory    │  │ - Labels    │    │
    │  │ - CXIMS     │  │ - MSI/MSI-X │  │   Tiers     │  │ - Security  │    │
    │  └─────┬───────┘  └─────┬───────┘  └─────┬───────┘  └─────┬───────┘    │
    └────────┼────────────────┼────────────────┼────────────────┼────────────┘
             │                │                │                │
    ┌────────┼────────────────┼────────────────┼────────────────┼────────────┐
    │        │                │                │                │            │
    │        ▼                ▼                ▼                ▼            │
    │                          Hardware Layer                                │
    │                                                                        │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                      CXL Host Bridge                           │   │
    │  │                                                                 │   │
    │  │  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐ │   │
    │  │  │ Root Complex  │────►│ CXL.io Logic  │────►│ CXL.mem Logic │ │   │
    │  │  │               │     │               │     │               │ │   │
    │  │  │ - PCIe Root   │     │ - PCI Express │     │ - Memory      │ │   │
    │  │  │   Port        │     │   Protocol    │     │   Protocol    │ │   │
    │  │  │ - ACPI        │     │ - Enhanced    │     │ - Coherent    │ │   │
    │  │  │   Integration │     │   Features    │     │   Memory      │ │   │
    │  │  └───────────────┘     └───────────────┘     └───────────────┘ │   │
    │  │                                                                 │   │
    │  │  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐ │   │
    │  │  │ HDM Decoders  │────►│ Root Decoders │────►│ CFMWS Windows │ │   │
    │  │  │               │     │               │     │               │ │   │
    │  │  │ - Address     │     │ - Capacity    │     │ - Memory      │ │   │
    │  │  │   Translation │     │   Discovery   │     │   Windows     │ │   │
    │  │  │ - Interleave  │     │ - Routing     │     │ - QoS         │ │   │
    │  │  │   Management  │     │   Rules       │     │   Classes     │ │   │
    │  │  └───────────────┘     └───────────────┘     └───────────────┘ │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                     │                                   │
    │                                     ▼                                   │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                       CXL Switch (Optional)                    │   │
    │  │                                                                 │   │
    │  │  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐ │   │
    │  │  │ Upstream Port │────►│ Switch Fabric │────►│Downstream Port│ │   │
    │  │  │               │     │               │     │               │ │   │
    │  │  │ - CXL.io      │     │ - Packet      │     │ - CXL.io      │ │   │
    │  │  │ - CXL.cache   │     │   Routing     │     │ - CXL.cache   │ │   │
    │  │  │ - CXL.mem     │     │ - Memory      │     │ - CXL.mem     │ │   │
    │  │  │               │     │   Coherency   │     │               │ │   │
    │  │  └───────────────┘     └───────────────┘     └───────────────┘ │   │
    │  │                                                                 │   │
    │  │  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐ │   │
    │  │  │ HDM Decoders  │────►│ Switch        │────►│ Port          │ │   │
    │  │  │               │     │ Management    │     │ Management    │ │   │
    │  │  │ - Decode      │     │               │     │               │ │   │
    │  │  │   Logic       │     │ - Health      │     │ - Link        │ │   │
    │  │  │ - Routing     │     │   Monitoring  │     │   Management  │ │   │
    │  │  │   Tables      │     │ - Error       │     │ - Power       │ │   │
    │  │  └───────────────┘     │   Handling    │     │   Management  │ │   │
    │  │                        └───────────────┘     └───────────────┘ │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                     │                                   │
    │                                     ▼                                   │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                      CXL Type-3 Device                         │   │
    │  │                                                                 │   │
    │  │  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐ │   │
    │  │  │ PCI Function  │────►│ CXL.io Logic  │────►│ CXL.mem Logic │ │   │
    │  │  │               │     │               │     │               │ │   │
    │  │  │ - PCI Express │     │ - Command     │     │ - Memory      │ │   │
    │  │  │   Endpoint    │     │   Interface   │     │   Interface   │ │   │
    │  │  │ - Config      │     │ - Security    │     │ - Coherent    │ │   │
    │  │  │   Registers   │     │   Features    │     │   Access      │ │   │
    │  │  └───────────────┘     └───────────────┘     └───────────────┘ │   │
    │  │                                                                 │   │
    │  │  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐ │   │
    │  │  │ Component     │────►│ Mailbox       │────►│ Memory        │ │   │
    │  │  │ Registers     │     │ Interface     │     │ Arrays        │ │   │
    │  │  │               │     │               │     │               │ │   │
    │  │  │ - HDM         │     │ - Command     │     │ - DDR/HBM     │ │   │
    │  │  │   Decoders    │     │   Processing  │     │ - Persistent  │ │   │
    │  │  │ - RAS         │     │ - Async       │     │   Memory      │ │   │
    │  │  │   Capability  │     │   Events      │     │ - NVM         │ │   │
    │  │  │ - Performance │     │ - Status      │     │ - Cache       │ │   │
    │  │  │   Monitoring  │     │   Reporting   │     │   Memory      │ │   │
    │  │  └───────────────┘     └───────────────┘     └───────────────┘ │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    └─────────────────────────────────────────────────────────────────────────┘

    Data Flow and Control Flow:
    
    1. Platform Discovery: cxl_acpi ←→ ACPI Namespace ←→ Host Bridge
    2. Device Discovery: cxl_pci ←→ PCI Config Space ←→ CXL Device  
    3. Memory Management: cxl_mem ←→ Memory Controller ←→ Memory Arrays
    4. Persistent Memory: cxl_pmem ←→ LIBNVDIMM ←→ NVM Arrays
    
    Communication Patterns:
    - MMIO for register access
    - Mailbox commands for device control
    - DMA for bulk data transfer
    - Interrupts for event notification
```

5. Module Collaboration Diagram
===============================

```
                        CXL Driver Module Collaboration

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         System Initialization                           │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                              cxl_core                                   │
    │  ┌───────────────────────────────────────────────────────────────────┐  │
    │  │ 1. Bus registration and core service initialization              │  │
    │  │    - Register cxl_bus_type                                       │  │  
    │  │    - Initialize device model                                     │  │
    │  │    - Setup core data structures                                  │  │
    │  └───────────────────────────────────────────────────────────────────┘  │
    └─────────────────┬───────────────────────────────────────────┬───────────┘
                      │                                           │
                      ▼                                           ▼
    ┌─────────────────────────────────┐           ┌─────────────────────────────────┐
    │           cxl_acpi              │           │            cxl_pci             │
    │                                 │           │                                 │
    │ ┌─────────────────────────────┐ │           │ ┌─────────────────────────────┐ │
    │ │ 2. Platform Discovery       │ │           │ │ 3. Device Discovery         │ │
    │ │    ┌─────────────────────┐  │ │           │ │    ┌─────────────────────┐  │ │
    │ │    │ Parse CEDT Table    │  │ │           │ │    │ PCI Device Probe    │  │ │
    │ │    │ - CHBS entries      │  │ │           │ │    │ - CXL Class Code    │  │ │
    │ │    │ - CFMWS entries     │  │ │           │ │    │ - Capability Check  │  │ │
    │ │    │ - CXIMS entries     │  │ │           │ │    └─────────────────────┘  │ │
    │ │    └─────────────────────┘  │ │           │ │            │                │ │
    │ │             │               │ │           │ │            ▼                │ │
    │ │    ┌─────────▼─────────────┐ │ │           │ │    ┌─────────────────────┐  │ │
    │ │    │ Create Root Ports     │ │ │           │ │    │ Register Mapping    │  │ │
    │ │    │ - Call cxl_core       │ │ │◄──────────┼─┼────┤ - Component regs    │  │ │
    │ │    │ - Port hierarchy      │ │ │           │ │    │ - Memory BARs       │  │ │
    │ │    └─────────────────────┐ │ │ │           │ │    └─────────────────────┘  │ │
    │ │                         │ │ │ │           │ │            │                │ │
    │ │    ┌────────────────────┼─┘ │ │           │ │            ▼                │ │
    │ │    │ Setup CFMWS        │   │ │           │ │    ┌─────────────────────┐  │ │
    │ │    │ - Memory windows   │   │ │           │ │    │ Mailbox Init        │  │ │
    │ │    │ - Root decoders    │   │ │           │ │    │ - Command interface │  │ │
    │ │    └────────────────────┘   │ │           │ │    │ - Interrupt setup   │  │ │
    │ └─────────────────────────────┘ │           │ │    └─────────────────────┘  │ │
    └─────────────────────────────────┘           │ │            │                │ │
                      │                           │ │            ▼                │ │
                      │                           │ │    ┌─────────────────────┐  │ │
                      │                           │ │    │ Create Memory       │  │ │
                      │                           │ │    │ Device              │  │ │
                      │                           │ │    │ - Call cxl_core     │  │ │
                      │                           │ │    └─────────────────────┘  │ │
                      │                           │ └─────────────────────────────┘ │
                      │                           └─────────────────────────────────┘
                      │                                           │
                      └─────────────┬─────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                              cxl_mem                                    │
    │                                                                         │
    │ ┌─────────────────────────────────────────────────────────────────────┐ │
    │ │ 4. Memory Configuration                                             │ │
    │ │    ┌─────────────────────┐         ┌─────────────────────┐          │ │
    │ │    │ Endpoint Creation   │         │ DPA Management      │          │ │
    │ │    │ - Port hierarchy    │         │ - Address spaces    │          │ │
    │ │    │ - Parent discovery  │         │ - Capacity tracking │          │ │
    │ │    └─────────────────────┘         └─────────────────────┘          │ │
    │ │             │                               │                       │ │
    │ │             └───────────────┬───────────────┘                       │ │
    │ │                             ▼                                       │ │
    │ │    ┌─────────────────────────────────────────────┐                  │ │
    │ │    │ HDM Decoder Configuration                   │                  │ │
    │ │    │ - Program decoders via cxl_core            │                  │ │
    │ │    │ - Setup address translation                 │                  │ │
    │ │    │ - Configure interleave parameters          │                  │ │
    │ │    └─────────────────────────────────────────────┘                  │ │
    │ └─────────────────────────────────────────────────────────────────────┘ │
    └─────────────────┬───────────────────────────────────────────┬───────────┘
                      │                                           │
                      ▼                                           ▼
    ┌─────────────────────────────────┐           ┌─────────────────────────────────┐
    │           cxl_pmem              │           │            cxl_port            │
    │                                 │           │                                 │
    │ ┌─────────────────────────────┐ │           │ ┌─────────────────────────────┐ │
    │ │ 5. Persistent Memory        │ │           │ │ 6. Port Operations          │ │
    │ │    ┌─────────────────────┐  │ │           │ │    ┌─────────────────────┐  │ │
    │ │    │ NVDIMM Bridge       │  │ │           │ │    │ Switch Management   │  │ │
    │ │    │ - LIBNVDIMM         │  │ │           │ │    │ - Port lifecycle    │  │ │
    │ │    │   integration       │  │ │           │ │    │ - Fabric operations │  │ │
    │ │    └─────────────────────┘  │ │           │ │    └─────────────────────┘  │ │
    │ │             │               │ │           │ │             │               │ │
    │ │             ▼               │ │           │ │             ▼               │ │
    │ │    ┌─────────────────────┐  │ │           │ │    ┌─────────────────────┐  │ │
    │ │    │ Namespace Support   │  │ │           │ │    │ Endpoint Management │  │ │
    │ │    │ - Create namespaces │  │ │           │ │    │ - Port-specific ops │  │ │
    │ │    │ - Label management  │  │ │           │ │    │ - Device binding    │  │ │
    │ │    └─────────────────────┘  │ │           │ │    └─────────────────────┘  │ │
    │ │             │               │ │           │ │                             │ │
    │ │             ▼               │ │           │ └─────────────────────────────┘ │
    │ │    ┌─────────────────────┐  │ │           └─────────────────────────────────┘
    │ │    │ Security Operations │  │ │                           │
    │ │    │ - Authentication    │  │ │                           │
    │ │    │ - Secure erase      │  │ │                           │
    │ │    └─────────────────────┘  │ │                           │
    │ └─────────────────────────────┘ │                           │
    └─────────────────────────────────┘                           │
                      │                                           │
                      └─────────────┬─────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        Runtime Operations                               │
    │                                                                         │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
    │  │ Region          │  │ Error           │  │ Performance     │          │
    │  │ Management      │  │ Handling        │  │ Monitoring      │          │
    │  │                 │  │                 │  │                 │          │
    │  │ - Memory region │  │ - RAS events    │  │ - PMU support   │          │
    │  │   creation      │  │ - Error         │  │ - Telemetry     │          │
    │  │ - Dynamic       │  │   injection     │  │ - QoS tracking  │          │
    │  │   reconfiguration│  │ - Recovery      │  │                 │          │
    │  └─────────────────┘  └─────────────────┘  └─────────────────┘          │
    │                                                                         │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
    │  │ Power           │  │ Security        │  │ Mailbox         │          │
    │  │ Management      │  │ Operations      │  │ Operations      │          │
    │  │                 │  │                 │  │                 │          │
    │  │ - Suspend/      │  │ - Key           │  │ - Command       │          │
    │  │   resume        │  │   management    │  │   processing    │          │
    │  │ - Dynamic       │  │ - Attestation   │  │ - Event         │          │
    │  │   power states  │  │ - Secure boot   │  │   handling      │          │
    │  └─────────────────┘  └─────────────────┘  └─────────────────┘          │
    └─────────────────────────────────────────────────────────────────────────┘

    Communication Patterns:
    1. cxl_acpi → cxl_core: Platform resource registration
    2. cxl_pci → cxl_core: Device registration and mailbox services  
    3. cxl_mem → cxl_core: Memory device creation and HDM configuration
    4. cxl_pmem → cxl_core: NVDIMM bridge registration
    5. cxl_port → cxl_core: Port operation callbacks
    6. All modules ← cxl_core: Common services (regions, decoders, etc.)
```

This comprehensive set of diagrams illustrates:
1. The overall framework architecture and layering
2. Module interactions and dependencies  
3. UEFI/firmware integration patterns
4. Hardware abstraction and communication
5. Collaborative workflows between modules

The diagrams show how the CXL driver framework provides a clean separation
between platform discovery (ACPI), device management (PCI), memory operations,
and integration with existing kernel subsystems.