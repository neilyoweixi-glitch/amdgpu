# AMDGPU Page Table Management Analysis

## Overview

This document analyzes how AMDGPU manages GPU page tables (GPUVM) in the Linux kernel driver.

## Page Table Architecture

### Hierarchy Structure

AMDGPU uses a multi-level page table hierarchy:

- **Levels**: 1-5 levels depending on ASIC family
  - Older ASICs: 1-2 levels
  - Newer ASICs (Vega10+): Up to 5 levels
- **Level Enumeration** (from `amdgpu_vm.h`):
  ```c
  enum amdgpu_vm_level {
      AMDGPU_VM_PDB2,  // Page Directory Block 2 (top level)
      AMDGPU_VM_PDB1,  // Page Directory Block 1
      AMDGPU_VM_PDB0,  // Page Directory Block 0
      AMDGPU_VM_PTB    // Page Table Block (leaf level)
  };
  ```

### Page Table Entry Format

Page table entries are 64-bit values. Key flags defined in `amdgpu_vm.h`:

#### Common Flags (across all ASICs):
- `AMDGPU_PTE_VALID` (bit 0): Entry is valid
- `AMDGPU_PTE_SYSTEM` (bit 1): Maps to system memory
- `AMDGPU_PTE_SNOOPED` (bit 2): Cache snooping enabled
- `AMDGPU_PTE_TMZ` (bit 3): Trusted Memory Zone (RV+)
- `AMDGPU_PTE_EXECUTABLE` (bit 4): Executable page (VI only)
- `AMDGPU_PTE_READABLE` (bit 5): Read permission
- `AMDGPU_PTE_WRITEABLE` (bit 6): Write permission
- `AMDGPU_PTE_FRAG(x)` (bits 7-11): Fragment size for large pages
- `AMDGPU_PTE_PRT` (bit 51): Partial Resident Texture (Vega10+)
- `AMDGPU_PDE_PTE` (bit 54): PDE handled as PTE (Vega10+)
- `AMDGPU_PTE_LOG` (bit 55): Logging enabled
- `AMDGPU_PTE_TF` (bit 56): Translate Further (Vega10+)
- `AMDGPU_PTE_NOALLOC` (bit 58): MALL noalloc (sienna_cichlid+)
- `AMDGPU_PDE_BFS(a)` (bits 59-63): PDE Block Fragment Size (Vega10+)

#### ASIC-Specific Flags:

**GFX9 (Vega10)**:
- `AMDGPU_PTE_MTYPE_VG10_SHIFT(mtype)` (bits 57-59): Memory type
  - `AMDGPU_MTYPE_NC`: Non-coherent
  - `AMDGPU_MTYPE_CC`: Coherent cacheable

**GFX10 (Navi)**:
- `AMDGPU_PTE_MTYPE_NV10_SHIFT(mtype)` (bits 48-54): Memory type

**GFX12**:
- `AMDGPU_PTE_PRT_GFX12` (bit 56): PRT flag at different position
- `AMDGPU_PTE_MTYPE_GFX12_SHIFT(mtype)` (bits 54-56): Memory type
- `AMDGPU_PTE_DCC` (bit 58): Delta Color Compression
- `AMDGPU_PTE_IS_PTE` (bit 63): Indicates this is a PTE entry
- `AMDGPU_PDE_BFS_GFX12(a)` (bits 58-62): PDE Block Fragment Size
- `AMDGPU_PDE_PTE_GFX12` (bit 63): PDE handled as PTE

### Physical Address Encoding

The physical address is encoded in the lower 48 bits of the PTE:
```c
value = addr & 0x0000FFFFFFFFF000ULL;  // 48-bit address, 4KB aligned
value |= flags;  // OR in the flags
```

## Page Table Management

### Key Files

1. **`amdgpu_vm.h`**: Core VM structures and PTE flag definitions
2. **`amdgpu_vm.c`**: Main VM management logic
3. **`amdgpu_vm_pt.c`**: Page table tree walking and management
4. **`amdgpu_vm_cpu.c`**: CPU-based page table updates
5. **`amdgpu_vm_sdma.c`**: SDMA-based page table updates
6. **`amdgpu_gmc.c`**: Graphics Memory Controller functions
7. **`amdgpu_gmc.h`**: GMC structures and function pointers

### Update Mechanisms

AMDGPU supports two methods for updating page tables:

#### 1. CPU Updates (`amdgpu_vm_cpu_funcs`)
- Direct CPU writes to page table memory
- Used when `vm->use_cpu_for_update` is true
- Functions:
  - `amdgpu_vm_cpu_map_table()`: Maps page table for CPU access
  - `amdgpu_vm_cpu_update()`: Writes PTE entries directly
  - `amdgpu_vm_cpu_commit()`: Flushes HDP cache

#### 2. SDMA Updates (`amdgpu_vm_sdma_funcs`)
- Uses System DMA engine to update page tables
- More efficient for bulk updates
- Functions:
  - `amdgpu_vm_sdma_map_table()`: Maps page table to GART
  - `amdgpu_vm_sdma_prepare()`: Allocates SDMA job
  - `amdgpu_vm_sdma_update()`: Writes commands to SDMA IB
  - `amdgpu_vm_sdma_commit()`: Submits SDMA job

### Page Table Tree Walking

The driver uses a cursor-based approach to walk the page table tree:

```c
struct amdgpu_vm_pt_cursor {
    uint64_t pfn;                    // Page frame number
    struct amdgpu_vm_bo_base *parent; // Parent directory
    struct amdgpu_vm_bo_base *entry;  // Current entry
    unsigned int level;              // Current level
};
```

Key functions:
- `amdgpu_vm_pt_start()`: Initialize cursor at root
- `amdgpu_vm_pt_descendant()`: Move to child node
- `amdgpu_vm_pt_sibling()`: Move to sibling node
- `amdgpu_vm_pt_ancestor()`: Move to parent node
- `amdgpu_vm_pt_next()`: Get next node in hierarchy

### Page Table Allocation

Page tables are allocated as buffer objects (BOs):

```c
int amdgpu_vm_pt_create(struct amdgpu_device *adev, struct amdgpu_vm *vm,
                       int level, bool immediate, struct amdgpu_bo_vm **vmbo,
                       int32_t xcp_id)
```

- Page tables are stored in VRAM (or GTT for APUs)
- Size depends on level:
  - Root level: Calculated based on `max_pfn`
  - Intermediate levels: 512 entries (9 bits)
  - Leaf level (PTB): `AMDGPU_VM_PTE_COUNT(adev)` entries
- Each entry is 8 bytes (64-bit)

### Fragment Support

AMDGPU supports variable-sized pages through fragments:

- Fragment size encoded in `AMDGPU_PTE_FRAG` field
- Allows mapping large contiguous ranges efficiently
- L1 TLB can cache a single PTE for the whole fragment
- Fragment size ranges from 0 (4KB) to 31 (up to 2GB pages on Vega10+)

## VMID and PASID

- **VMID**: Virtual Memory ID (0-15), hardware identifier for active VMs
- **PASID**: Process Address Space ID, used for per-process VMs
- VMID 0 is special: kernel driver's VM with direct VRAM/AGP apertures
- Each VM can have multiple VMIDs across different VMHUBs (GFXHUB, MMHUB)

## VMHUBs

AMDGPU supports multiple VMHUBs:
- **GFXHUB**: Graphics hub (up to 8 instances)
- **MMHUB0**: Multi-media hub 0 (up to 4 instances)
- **MMHUB1**: Multi-media hub 1 (1 instance)
- Each hub has its own page table base address registers

## Documentation Status

### Existing Documentation

1. **Kernel Documentation** (`Documentation/gpu/amdgpu/amdgpu-glossary.rst`):
   - Defines GPUVM terminology
   - Explains basic concepts
   - No detailed page table format specification

2. **Code Comments** (`amdgpu_vm.c`):
   - DOC: GPUVM section explains high-level architecture
   - Describes 1-5 level page tables
   - Mentions RWX attributes and encryption

### Missing Documentation

**There is NO comprehensive documentation explaining:**
- Detailed bit-level PTE format for each ASIC generation
- PDE format differences between ASICs
- Exact encoding of physical addresses
- Fragment size calculation details
- Memory type encoding specifics
- Page table walker behavior
- TLB invalidation mechanisms

### Recommendations

To document GPU page table formats, consider:

1. **Create a new documentation file**: `Documentation/gpu/amdgpu/page-table-formats.rst`
   - Document PTE bit layouts for each ASIC generation
   - Include PDE formats
   - Explain fragment encoding
   - Provide examples

2. **Add inline documentation**:
   - Document `amdgpu_gmc_set_pte_pde()` with bit field details
   - Add comments to PTE flag definitions explaining bit positions
   - Document ASIC-specific differences

3. **Reference Hardware Documentation**:
   - Link to AMD GPU architecture documentation (if available)
   - Note which registers control page table behavior

## Key Implementation Details

### Page Table Updates

1. **Range Updates**: `amdgpu_vm_update_range()`
   - Updates a contiguous range of PTEs
   - Handles fragment boundaries
   - Supports both CPU and SDMA updates

2. **PDE Updates**: `amdgpu_vm_pde_update()`
   - Updates parent directory entries
   - Points to child page tables
   - Handles level-specific encoding

3. **PTE Updates**: `amdgpu_vm_ptes_update()`
   - Core function for updating PTEs
   - Handles fragment calculation
   - Walks page table tree
   - Updates entries at appropriate level

### TLB Management

- TLB flushes required after page table updates
- Sequence numbers track TLB state
- Different flush types for different scenarios
- Hardware-specific flush mechanisms

## Conclusion

AMDGPU uses a sophisticated multi-level page table system with:
- Flexible hierarchy (1-5 levels)
- Variable page sizes (fragments)
- Multiple update mechanisms (CPU/SDMA)
- ASIC-specific encoding differences

However, **there is currently no comprehensive documentation explaining the detailed GPU page table formats**. The format details are embedded in the code through flag definitions and ASIC-specific functions, but a formal specification document would be valuable for developers and researchers.
