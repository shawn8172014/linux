===================================
CXL Driver UEFI and Firmware Integration
===================================

:Author: Linux CXL Subsystem Analysis  
:Date: 2024

Overview
========

This document provides detailed analysis of how the CXL driver framework
integrates with UEFI firmware and ACPI to enable CXL device discovery,
configuration, and management. The integration spans multiple layers from
boot-time platform discovery to runtime device management.

UEFI/ACPI Integration Architecture
=================================

The CXL driver integrates with firmware through several standardized interfaces:

1. **UEFI Boot Services**: Initial platform discovery during boot
2. **ACPI Tables**: Persistent platform configuration data
3. **ACPI Namespace**: Runtime device and resource management
4. **PCIe Configuration**: Standard PCI enumeration enhanced for CXL

ACPI Table Integration
=====================

1. CEDT (CXL Early Discovery Table)
-----------------------------------

The CEDT table is the primary mechanism for early CXL platform discovery.

**CEDT Structure Overview**::

    CEDT (CXL Early Discovery Table)
    ├── Header (Standard ACPI Table Header)
    ├── CHBS (CXL Host Bridge Structure) entries
    ├── CFMWS (CXL Fixed Memory Window Structure) entries  
    ├── CXIMS (CXL XOR Interleave Math Structure) entries
    └── Additional CXL structures (future extensions)

**CEDT Parsing Flow**::

    ┌─────────────────┐
    │ UEFI/BIOS       │
    │ Creates CEDT    │
    │ during POST     │
    └─────────┬───────┘
              │
              ▼
    ┌─────────────────┐
    │ ACPI Core       │
    │ Provides CEDT   │
    │ to OS           │
    └─────────┬───────┘
              │
              ▼
    ┌─────────────────┐
    │ cxl_acpi        │
    │ - acpi_probe()  │
    │ - cedt_parse()  │
    └─────────┬───────┘
              │
              ▼
    ┌─────────────────┐
    │ Platform        │
    │ Configuration   │
    │ Complete        │
    └─────────────────┘

2. CHBS (CXL Host Bridge Structure)
-----------------------------------

CHBS entries describe CXL-capable host bridges in the system.

**CHBS Structure Fields**::

    struct acpi_cedt_chbs {
        struct acpi_cedt_header header;
        u32 uid;                    /* Unique ID */
        u32 cxl_version;           /* CXL specification version */  
        u32 rcrb_base_high;        /* Register access base (high) */
        u32 rcrb_base_low;         /* Register access base (low) */
        u32 length;                /* Length of register space */
    };

**CHBS Processing in cxl_acpi.c**::

    static int cxl_parse_chbs(union acpi_subtable_headers *header,
                             void *arg, const unsigned long end)
    {
        struct acpi_cedt_chbs *chbs = (struct acpi_cedt_chbs *)header;
        struct cxl_chbs_context *ctx = arg;
        
        /* Extract host bridge information */
        ctx->uid = chbs->uid;
        ctx->rcrb_base = ((u64)chbs->rcrb_base_high << 32) | 
                        chbs->rcrb_base_low;
        
        /* Create CXL root port */
        return cxl_acpi_add_root_port(ctx);
    }

3. CFMWS (CXL Fixed Memory Window Structure)  
--------------------------------------------

CFMWS entries define memory address ranges that can be used by CXL devices.

**CFMWS Structure Fields**::

    struct acpi_cedt_cfmws {
        struct acpi_cedt_header header;
        u32 reserved1;
        u64 base_hpa;              /* Base Host Physical Address */
        u64 window_size;           /* Size of memory window */
        u8 interleave_ways;        /* Number of interleave ways */
        u8 interleave_arithmetic;  /* Interleave algorithm */
        u16 reserved2;
        u32 granularity;           /* Interleave granularity */
        u16 restrictions;          /* Usage restrictions */
        u16 qtg_id;               /* QoS Throttling Group ID */
        u32 interleave_targets[]; /* Target list (variable) */
    };

**CFMWS Processing Flow**::

    CFMWS Entry
        │
        ▼
    ┌─────────────────────────────────┐
    │ cxl_parse_cfmws()               │
    │ - Extract memory window info    │
    │ - Validate interleave params    │
    │ - Check QoS requirements        │
    └─────────────┬───────────────────┘
                  │
                  ▼
    ┌─────────────────────────────────┐
    │ cxl_decoder_alloc()             │
    │ - Create root decoder object    │
    │ - Configure address range       │
    │ - Set interleave parameters     │
    └─────────────┬───────────────────┘
                  │
                  ▼
    ┌─────────────────────────────────┐
    │ cxl_root_decoder_populate()     │
    │ - Setup target routing          │
    │ - Configure QoS classes         │
    │ - Enable memory window          │
    └─────────────────────────────────┘

4. CXIMS (CXL XOR Interleave Math Structure)
--------------------------------------------

CXIMS entries provide XOR-based interleave mathematics for complex topologies.

**CXIMS Structure and Processing**::

    struct acpi_cedt_cxims {
        struct acpi_cedt_header header;
        u8 hbig;                   /* Host Bridge Interleave Granularity */
        u8 nr_xormaps;            /* Number of XOR maps */
        u16 reserved;
        u64 xormaps[];            /* XOR map array */
    };

    /* XOR interleave processing */
    static u64 cxl_xor_hpa_to_spa(struct cxl_root_decoder *cxlrd, u64 hpa)
    {
        struct cxl_cxims_data *cximsd = cxlrd->platform_data;
        
        for (int i = 0; i < cximsd->nr_maps; i++) {
            /* Apply XOR transformation */
            pos = __ffs(cximsd->xormaps[i]);
            val = (hweight64(hpa & cximsd->xormaps[i]) & 1);
            hpa = (hpa & ~(1ULL << pos)) | (val << pos);
        }
        
        return hpa;
    }

ACPI Runtime Integration
=======================

1. ACPI Namespace Integration
-----------------------------

**Device Object Hierarchy**::

    ACPI Namespace
    ├── \_SB (System Bus)
    │   ├── PCI0 (PCI Root Complex)
    │   │   ├── CXL0 (CXL Host Bridge)
    │   │   │   ├── _HID: ACPI0016 
    │   │   │   ├── _CID: Compatible ID
    │   │   │   ├── _UID: Unique ID
    │   │   │   └── _DSD: Device Specific Data
    │   │   └── Device-specific methods
    │   └── Memory topology nodes

**ACPI Method Integration**::

    /* Example ACPI methods used by CXL driver */
    _OSC    /* Operating System Capabilities */
    _DSM    /* Device Specific Method */
    _DSD    /* Device Specific Data */
    _PLD    /* Physical Location Description */
    _DDN    /* DOS Device Name */

2. NUMA Topology Integration
----------------------------

**NUMA Integration Flow**::

    ┌─────────────────┐
    │ ACPI SRAT       │
    │ (Proximity      │
    │  Domains)       │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ ACPI HMAT       │  
    │ (Memory         │
    │  Performance)   │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ cxl_acpi        │
    │ NUMA Node       │
    │ Assignment      │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ Memory Tier     │
    │ Classification  │
    └─────────────────┘

**NUMA Node Assignment Code**::

    static int cxl_acpi_setup_numa(struct cxl_port *root_port,
                                   struct acpi_cedt_cfmws *cfmws)
    {
        int target, node;
        
        for (target = 0; target < cfmws->interleave_ways; target++) {
            /* Get NUMA node for each target */
            node = acpi_get_node(cfmws->interleave_targets[target]);
            if (node == NUMA_NO_NODE) {
                /* Assign default node */
                node = numa_mem_id(cfmws->base_hpa);
            }
            
            /* Configure memory tier */
            cxl_assign_memory_tier(root_port, target, node);
        }
        
        return 0;
    }

3. QoS and Performance Integration
---------------------------------

**QoS Data Flow**::

    ACPI HMAT Tables
           │
           ▼
    ┌─────────────────┐
    │ QTG (QoS        │
    │ Throttling      │  
    │ Group) ID       │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ CDAT (Coherent  │
    │ Device Attribute│
    │ Table)          │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ Performance     │
    │ Characteristics │
    │ Database        │
    └─────────────────┘

**QoS Processing Implementation**::

    struct cxl_dpa_perf {
        int dpa_range_start;
        int dpa_range_end;
        int qos_class;
        struct access_coordinate coord;
    };

    static int cxl_parse_qtg_id(struct cxl_root_decoder *cxlrd,
                                struct acpi_cedt_cfmws *cfmws)
    {
        /* Extract QoS Throttling Group ID */
        cxlrd->qtg_id = cfmws->qtg_id;
        
        /* Query HMAT for performance data */
        return acpi_get_memory_performance(cfmws->base_hpa,
                                         cfmws->window_size,
                                         &cxlrd->coord);
    }

UEFI Runtime Services Integration
===============================

1. Variable Services
--------------------

**CXL-Related UEFI Variables**::

    /* Example UEFI variables that may affect CXL */
    CxlConfiguration    /* Global CXL settings */
    CxlDevicePolicy     /* Per-device policies */
    CxlMemoryMap        /* Memory layout information */
    CxlSecurityState    /* Security configuration */

**Variable Access Pattern**::

    static int cxl_read_uefi_config(struct cxl_port *port)
    {
        efi_guid_t cxl_guid = CXL_CONFIGURATION_GUID;
        unsigned long size;
        void *data;
        
        /* Read CXL configuration from UEFI */
        efi_status = efi.get_variable(L"CxlConfiguration",
                                     &cxl_guid, NULL, &size, data);
        
        if (efi_status == EFI_SUCCESS) {
            /* Apply configuration */
            return cxl_apply_uefi_config(port, data, size);
        }
        
        return 0; /* Use defaults */
    }

2. Memory Map Integration
-------------------------

**UEFI Memory Map and CXL**::

    ┌─────────────────┐
    │ UEFI Memory Map │
    │ - Reserved      │
    │ - ACPI NVS      │
    │ - MMIO          │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ E820 Memory Map │
    │ (x86 specific)  │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ CXL Memory      │
    │ Integration     │
    │ - Validate      │
    │   ranges        │
    │ - Reserve       │
    │   regions       │
    └─────────────────┘

Platform Security Integration
============================

1. Secure Boot Integration
--------------------------

**Secure Boot Flow**::

    ┌─────────────────┐
    │ UEFI Secure     │
    │ Boot Validation │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ Kernel Module   │
    │ Signature       │  
    │ Verification    │
    └─────┬───────────┘
          │
          ▼
    ┌─────────────────┐
    │ CXL Driver      │
    │ Authentication  │
    │ - Device certs  │
    │ - Secure comms  │
    └─────────────────┘

2. TPM Integration
------------------

**TPM-Based Security**::

    struct cxl_security_state {
        bool secure_boot_enabled;
        bool tpm_available;
        u32 pcr_index;              /* TPM PCR for CXL measurements */
        struct tpm_chip *tpm;       /* TPM device handle */
    };

    static int cxl_extend_tpm_pcr(struct cxl_memdev *cxlmd, 
                                  const u8 *measurement)
    {
        struct tpm_digest digest = {
            .alg_id = TPM_ALG_SHA256,
            .digest = measurement
        };
        
        return tpm_pcr_extend(cxlmd->security.tpm, 
                             cxlmd->security.pcr_index, &digest);
    }

Firmware Interface Implementation
================================

1. ACPI Driver Implementation (cxl_acpi.c)
------------------------------------------

**Key Implementation Functions**::

    static int cxl_acpi_probe(struct platform_device *pdev)
    {
        struct acpi_table_cedt *cedt;
        acpi_status status;
        int rc;
        
        /* Get CEDT table */
        status = acpi_get_table(ACPI_SIG_CEDT, 0, 
                               (struct acpi_table_header **)&cedt);
        if (ACPI_FAILURE(status))
            return -ENODEV;
            
        /* Parse CEDT entries */
        rc = acpi_table_parse_entries(ACPI_SIG_CEDT,
                                     sizeof(struct acpi_table_cedt),
                                     ACPI_CEDT_TYPE_CHBS,
                                     cxl_parse_chbs, 0);
        
        rc = acpi_table_parse_entries(ACPI_SIG_CEDT,
                                     sizeof(struct acpi_table_cedt), 
                                     ACPI_CEDT_TYPE_CFMWS,
                                     cxl_parse_cfmws, 0);
        
        return rc;
    }

**CFMWS Parser Implementation**::

    static int cxl_parse_cfmws(union acpi_subtable_headers *header,
                              void *arg, const unsigned long end)
    {
        struct acpi_cedt_cfmws *cfmws = (struct acpi_cedt_cfmws *)header;
        struct cxl_cfmws_context *ctx = arg;
        struct cxl_decoder *cxld;
        int rc;
        
        /* Validate CFMWS entry */
        if (cfmws->header.length < sizeof(*cfmws))
            return -EINVAL;
            
        /* Create root decoder */
        cxld = cxl_root_decoder_alloc(ctx->root_port, cfmws->interleave_ways);
        if (IS_ERR(cxld))
            return PTR_ERR(cxld);
            
        /* Configure decoder parameters */
        cxld->base = cfmws->base_hpa;
        cxld->size = cfmws->window_size;
        cxld->interleave_ways = cfmws->interleave_ways;
        cxld->interleave_granularity = cfmws->granularity;
        
        /* Setup interleave targets */
        for (int i = 0; i < cfmws->interleave_ways; i++) {
            cxld->targets[i] = cfmws->interleave_targets[i];
        }
        
        /* Register decoder */
        rc = cxl_decoder_add_locked(cxld, cfmws->interleave_targets);
        if (rc)
            put_device(&cxld->dev);
            
        return rc;
    }

2. Platform Device Integration
------------------------------

**Platform Driver Structure**::

    static const struct acpi_device_id cxl_acpi_ids[] = {
        { "ACPI0017" },  /* CXL namespace device */
        { }
    };
    MODULE_DEVICE_TABLE(acpi, cxl_acpi_ids);

    static struct platform_driver cxl_acpi_driver = {
        .probe = cxl_acpi_probe,
        .remove = cxl_acpi_remove,
        .driver = {
            .name = "cxl_acpi",
            .acpi_match_table = cxl_acpi_ids,
        },
    };

Error Handling and Recovery
===========================

1. ACPI Error Handling
----------------------

**Error Recovery Patterns**::

    static int cxl_acpi_handle_error(struct cxl_port *port, 
                                    acpi_status status)
    {
        switch (status) {
        case AE_NOT_FOUND:
            /* CEDT not present - PCI-only operation */
            dev_info(&port->dev, "No CEDT found, using PCI discovery\n");
            return cxl_pci_only_init(port);
            
        case AE_BAD_DATA:
            /* Corrupted ACPI data */
            dev_err(&port->dev, "Corrupted CEDT data\n");
            return -EINVAL;
            
        case AE_NO_MEMORY:
            /* Resource exhaustion */
            return -ENOMEM;
            
        default:
            /* Unknown error */
            dev_err(&port->dev, "ACPI error: %s\n", 
                   acpi_format_exception(status));
            return -EIO;
        }
    }

2. Graceful Degradation
-----------------------

**Fallback Mechanisms**::

    static int cxl_platform_init(void)
    {
        int rc;
        
        /* Try ACPI-based initialization first */
        rc = cxl_acpi_init();
        if (rc == 0)
            return 0;  /* Success */
            
        /* Fall back to PCI-only discovery */
        dev_warn(cxl_bus_dev, "ACPI init failed, using PCI-only mode\n");
        return cxl_pci_only_init();
    }

Debug and Diagnostics
====================

1. ACPI Debug Support
---------------------

**Debug Information Export**::

    static int cxl_acpi_debug_show(struct seq_file *m, void *v)
    {
        struct acpi_table_cedt *cedt;
        
        seq_printf(m, "CXL ACPI Debug Information\n");
        seq_printf(m, "CEDT Address: %p\n", cedt);
        seq_printf(m, "CEDT Length: %u\n", cedt->header.length);
        seq_printf(m, "CEDT Revision: %u\n", cedt->header.revision);
        
        /* Dump CEDT entries */
        cxl_acpi_dump_cedt_entries(m, cedt);
        
        return 0;
    }

2. Firmware Version Compatibility
---------------------------------

**Version Checking**::

    static bool cxl_check_firmware_compatibility(struct acpi_table_cedt *cedt)
    {
        /* Check CEDT revision */
        if (cedt->header.revision < CXL_MIN_CEDT_REVISION) {
            pr_warn("CEDT revision %u too old, minimum %u required\n",
                   cedt->header.revision, CXL_MIN_CEDT_REVISION);
            return false;
        }
        
        /* Check for required entries */
        if (!cxl_cedt_has_required_entries(cedt)) {
            pr_warn("CEDT missing required entries\n");
            return false;
        }
        
        return true;
    }

Future Firmware Integration
==========================

1. Enhanced ACPI Support
------------------------

**Future ACPI Extensions**:
  - Dynamic CXL device hotplug via ACPI
  - Enhanced QoS configuration
  - Security policy enforcement
  - Power management coordination

2. UEFI Runtime Enhancement
--------------------------

**Planned Enhancements**:
  - Runtime CXL configuration updates
  - Firmware-assisted error recovery
  - Enhanced security attestation
  - Performance optimization coordination

Conclusion
==========

The CXL driver's UEFI and firmware integration provides:

- **Comprehensive Platform Discovery**: Through ACPI CEDT parsing
- **Runtime Configuration Management**: Via ACPI namespace integration  
- **Security Integration**: With UEFI Secure Boot and TPM
- **Performance Optimization**: Through QoS and NUMA integration
- **Error Recovery**: With graceful degradation mechanisms
- **Future Extensibility**: Supporting evolving firmware capabilities

This integration enables CXL devices to be seamlessly integrated into platform
firmware architecture while maintaining compatibility with existing systems
and providing a foundation for future enhancements.