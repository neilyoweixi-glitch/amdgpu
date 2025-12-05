# AMDGPU Virtual Memory Management - File-Level Analysis

## Overview

This document provides a comprehensive breakdown of all files related to AMDGPU virtual memory management, including both C source files (.c) and header files (.h), with line-of-code statistics.

---

## File Categories

### 1. Core VM Management Files

#### 1.1 Main VM Implementation

**`amdgpu_vm.c`** (3,285 lines)
- **Purpose**: Core virtual memory management implementation
- **Key Functions**:
  - `amdgpu_vm_init()` - Initialize VM
  - `amdgpu_vm_bo_map()` - Map buffer object to virtual address
  - `amdgpu_vm_bo_unmap()` - Unmap buffer object
  - `amdgpu_vm_update_range()` - Update page table entries for a range
  - `amdgpu_vm_handle_fault()` - Handle GPU page faults
  - `amdgpu_vm_bo_update()` - Update all BO mappings in VM
- **Evidence**: Lines 1-3285
- **Code**: ~1,944 lines | **Comments**: ~909 lines | **Blanks**: ~432 lines

**`amdgpu_vm.h`** (710 lines)
- **Purpose**: VM data structures and function declarations
- **Key Structures**:
  - `struct amdgpu_vm` - Main VM structure
  - `struct amdgpu_vm_manager` - VM manager
  - `struct amdgpu_bo_va` - BO-VM association
  - `struct amdgpu_bo_va_mapping` - VA range mapping
  - `struct amdgpu_vm_update_params` - Update parameters
- **Evidence**: Lines 1-710
- **Code**: ~401 lines | **Comments**: ~188 lines | **Blanks**: ~121 lines

**Total Core VM**: **3,995 lines** (2,345 code + 1,097 comments + 553 blanks)

---

### 2. Page Table Operations

**`amdgpu_vm_pt.c`** (970 lines)
- **Purpose**: Page table tree walking and manipulation
- **Key Functions**:
  - `amdgpu_vm_ptes_update()` - Update PTE entries
  - `amdgpu_vm_pde_update()` - Update PDE entries
  - `amdgpu_vm_pt_create()` - Create page table BO
  - `amdgpu_vm_pt_clear()` - Clear page table entries
  - `amdgpu_vm_pte_fragment()` - Calculate fragment size
  - Tree walking: `amdgpu_vm_pt_start()`, `amdgpu_vm_pt_descendant()`, `amdgpu_vm_pt_next()`
- **Evidence**: Lines 1-970
- **Code**: ~491 lines | **Comments**: ~366 lines | **Blanks**: ~113 lines

---

### 3. Page Table Update Mechanisms

#### 3.1 CPU-Based Updates

**`amdgpu_vm_cpu.c`** (122 lines)
- **Purpose**: CPU direct memory writes for page table updates
- **Key Functions**:
  - `amdgpu_vm_cpu_map_table()` - Map page table for CPU access
  - `amdgpu_vm_cpu_update()` - Write PTE entries directly
  - `amdgpu_vm_cpu_commit()` - Flush HDP cache
- **Evidence**: Lines 1-122
- **Code**: ~54 lines | **Comments**: ~56 lines | **Blanks**: ~12 lines

#### 3.2 SDMA-Based Updates

**`amdgpu_vm_sdma.c`** (296 lines)
- **Purpose**: SDMA engine-based page table updates
- **Key Functions**:
  - `amdgpu_vm_sdma_map_table()` - Map page table to GART
  - `amdgpu_vm_sdma_update()` - Write commands to SDMA IB
  - `amdgpu_vm_sdma_commit()` - Submit SDMA job
  - `amdgpu_vm_sdma_set_ptes()` - Setup PTE update commands
  - `amdgpu_vm_sdma_copy_ptes()` - Copy PTE entries
- **Evidence**: Lines 1-296
- **Code**: ~163 lines | **Comments**: ~94 lines | **Blanks**: ~39 lines

**Total Update Mechanisms**: **418 lines** (217 code + 150 comments + 51 blanks)

---

### 4. TLB Management

**`amdgpu_vm_tlb_fence.c`** (111 lines)
- **Purpose**: TLB flush fence management
- **Key Functions**:
  - TLB flush sequence tracking
  - Fence callbacks for TLB invalidation
- **Evidence**: Lines 1-111
- **Code**: ~64 lines | **Comments**: ~29 lines | **Blanks**: ~18 lines

---

### 5. Graphics Memory Controller (GMC) Core

**`amdgpu_gmc.c`** (1,692 lines)
- **Purpose**: Graphics Memory Controller core functions
- **Key Functions**:
  - `amdgpu_gmc_set_pte_pde()` - Write PTE/PDE entries
  - `amdgpu_gmc_get_pde_for_bo()` - Get PDE for buffer object
  - `amdgpu_gmc_pd_addr()` - Get page directory address
  - `amdgpu_gmc_filter_faults()` - Filter duplicate faults
  - `amdgpu_gmc_vram_location()` - Setup VRAM location
  - `amdgpu_gmc_gart_location()` - Setup GART location
- **Evidence**: Lines 1-1692
- **Code**: ~1,134 lines | **Comments**: ~337 lines | **Blanks**: ~221 lines

**`amdgpu_gmc.h`** (471 lines)
- **Purpose**: GMC structures and function pointers
- **Key Structures**:
  - `struct amdgpu_gmc` - GMC configuration
  - `struct amdgpu_gmc_funcs` - Function pointers for GMC operations
  - `struct amdgpu_vmhub` - VMHUB configuration
- **Evidence**: Lines 1-471
- **Code**: ~295 lines | **Comments**: ~127 lines | **Blanks**: ~49 lines

**Total GMC Core**: **2,163 lines** (1,429 code + 464 comments + 270 blanks)

---

### 6. GMC Version-Specific Implementations

These files implement GMC functionality for specific GPU generations:

#### GFX6 (Southern Islands)
- **`gmc_v6_0.c`** (1,171 lines) - Code: ~916 | Comments: ~70 | Blanks: ~185
- **`gmc_v6_0.h`** (29 lines) - Code: ~4 | Comments: ~22 | Blanks: ~3

#### GFX7 (Sea Islands)
- **`gmc_v7_0.c`** (1,399 lines) - Code: ~1,019 | Comments: ~180 | Blanks: ~200
- **`gmc_v7_0.h`** (30 lines) - Code: ~5 | Comments: ~22 | Blanks: ~3

#### GFX8 (Volcanic Islands)
- **`gmc_v8_0.c`** (1,783 lines) - Code: ~1,295 | Comments: ~228 | Blanks: ~260
- **`gmc_v8_0.h`** (31 lines) - Code: ~6 | Comments: ~22 | Blanks: ~3

#### GFX9 (Vega)
- **`gmc_v9_0.c`** (2,381 lines) - Code: ~1,758 | Comments: ~340 | Blanks: ~283
- **`gmc_v9_0.h`** (31 lines) - Code: ~6 | Comments: ~22 | Blanks: ~3
- **Key Features**: VMC interrupt handling, page fault processing

#### GFX10 (Navi/RDNA)
- **`gmc_v10_0.c`** (1,152 lines) - Code: ~734 | Comments: ~237 | Blanks: ~181
- **`gmc_v10_0.h`** (30 lines) - Code: ~5 | Comments: ~22 | Blanks: ~3

#### GFX11 (RDNA2/RDNA3)
- **`gmc_v11_0.c`** (1,054 lines) - Code: ~673 | Comments: ~216 | Blanks: ~165
- **`gmc_v11_0.h`** (30 lines) - Code: ~5 | Comments: ~22 | Blanks: ~3

#### GFX12 (Latest)
- **`gmc_v12_0.c`** (1,043 lines) - Code: ~648 | Comments: ~225 | Blanks: ~170
- **`gmc_v12_0.h`** (30 lines) - Code: ~5 | Comments: ~22 | Blanks: ~3

**Total GMC Version-Specific**: **10,194 lines** (7,483 code + 1,551 comments + 1,160 blanks)

**Key GMC Functions** (from `gmc_v9_0.c`):
- `gmc_v9_0_process_interrupt()` - Process VMC interrupts (line 565)
- `gmc_v9_0_vm_fault_interrupt_state()` - Enable/disable fault interrupts (line 463)
- `gmc_v9_0_flush_gpu_tlb()` - Flush TLB (line 753)
- `gmc_v9_0_set_pte_pde()` - ASIC-specific PTE/PDE writing

---

### 7. Buffer Object Management

**`amdgpu_object.c`** (1,735 lines)
- **Purpose**: Buffer object lifecycle management
- **Key Functions**:
  - `amdgpu_bo_create()` - Create buffer object
  - `amdgpu_bo_pin()` - Pin BO in memory
  - `amdgpu_bo_kmap()` - Map BO for CPU access
  - `amdgpu_bo_gpu_offset()` - Get GPU address
  - `amdgpu_bo_move_notify()` - Handle BO moves
- **Evidence**: Lines 1-1735
- **Code**: ~991 lines | **Comments**: ~512 lines | **Blanks**: ~232 lines

**`amdgpu_object.h`** (374 lines)
- **Purpose**: BO data structures
- **Key Structures**:
  - `struct amdgpu_bo` - Main buffer object
  - `struct amdgpu_bo_va` - BO-VM association
  - `struct amdgpu_bo_va_mapping` - VA mapping
  - `struct amdgpu_bo_user` - User-space BO
  - `struct amdgpu_bo_vm` - Page table BO
- **Evidence**: Lines 1-374
- **Code**: ~251 lines | **Comments**: ~81 lines | **Blanks**: ~42 lines

**Total Buffer Objects**: **2,109 lines** (1,242 code + 593 comments + 274 blanks)

---

### 8. TTM Integration

**`amdgpu_ttm.c`** (3,278 lines)
- **Purpose**: TTM (Translation Table Manager) integration
- **Key Functions**:
  - TTM BO management
  - Memory domain management
  - VRAM/GTT allocation
  - BO validation and eviction
- **Evidence**: Lines 1-3278
- **Code**: ~2,189 lines | **Comments**: ~594 lines | **Blanks**: ~495 lines

**`amdgpu_ttm.h`** (275 lines)
- **Purpose**: TTM-related structures and definitions
- **Evidence**: Lines 1-275
- **Code**: ~210 lines | **Comments**: ~30 lines | **Blanks**: ~35 lines

**Total TTM**: **3,553 lines** (2,399 code + 624 comments + 530 blanks)

---

### 9. KFD (Kernel Fusion Driver) Integration

**`amdgpu_amdkfd_gpuvm.c`** (3,674 lines)
- **Purpose**: Integration with KFD for compute contexts
- **Key Functions**:
  - SVM (Shared Virtual Memory) support
  - Compute context VM management
  - Page migration for compute
- **Evidence**: Lines 1-3674
- **Code**: ~2,528 lines | **Comments**: ~632 lines | **Blanks**: ~514 lines

---

## Summary Statistics

### By Category

| Category | Files | Total Lines | Code Lines | Comment Lines | Blank Lines |
|----------|-------|-------------|------------|---------------|-------------|
| **Core VM Management** | 2 | 3,995 | 2,345 | 1,097 | 553 |
| **Page Table Operations** | 1 | 970 | 491 | 366 | 113 |
| **CPU Updates** | 1 | 122 | 54 | 56 | 12 |
| **SDMA Updates** | 1 | 296 | 163 | 94 | 39 |
| **TLB Management** | 1 | 111 | 64 | 29 | 18 |
| **GMC Core** | 2 | 2,163 | 1,429 | 464 | 270 |
| **GMC Version-Specific** | 14 | 10,194 | 7,483 | 1,551 | 1,160 |
| **Buffer Objects** | 2 | 2,109 | 1,242 | 593 | 274 |
| **TTM Integration** | 2 | 3,553 | 2,399 | 624 | 530 |
| **KFD Integration** | 1 | 3,674 | 2,528 | 632 | 514 |
| **TOTAL** | **27** | **27,187** | **18,198** | **5,506** | **3,483** |

### File Size Distribution

**Large Files (>2000 lines)**:
1. `amdgpu_vm.c` - 3,285 lines
2. `amdgpu_amdkfd_gpuvm.c` - 3,674 lines
3. `amdgpu_ttm.c` - 3,278 lines
4. `gmc_v9_0.c` - 2,381 lines

**Medium Files (1000-2000 lines)**:
1. `gmc_v8_0.c` - 1,783 lines
2. `amdgpu_object.c` - 1,735 lines
3. `amdgpu_gmc.c` - 1,692 lines
4. `gmc_v7_0.c` - 1,399 lines
5. `gmc_v6_0.c` - 1,171 lines
6. `gmc_v10_0.c` - 1,152 lines
7. `gmc_v11_0.c` - 1,054 lines
8. `gmc_v12_0.c` - 1,043 lines

**Small Files (<1000 lines)**:
- All header files and specialized implementation files

---

## File Dependencies

### Core Dependencies

```
amdgpu_vm.c
├── amdgpu_vm.h (definitions)
├── amdgpu_vm_pt.c (page table operations)
├── amdgpu_vm_cpu.c (CPU updates)
├── amdgpu_vm_sdma.c (SDMA updates)
├── amdgpu_gmc.c (GMC functions)
├── amdgpu_object.c (buffer objects)
└── amdgpu_ttm.c (memory management)

amdgpu_gmc.c
├── amdgpu_gmc.h (definitions)
├── gmc_v[6-12]_0.c (version-specific)
└── amdgpu_vm.h (VM structures)

amdgpu_object.c
├── amdgpu_object.h (definitions)
├── amdgpu_vm.h (VM-BO structures)
└── amdgpu_ttm.h (TTM structures)
```

---

## Related Files (Indirect Dependencies)

### VMHUB Files (VM Hub Management)

**GFXHUB Files** (Graphics Hub):
- `gfxhub_v1_0.c`, `gfxhub_v1_0.h`
- `gfxhub_v1_1.c`, `gfxhub_v1_1.h`
- `gfxhub_v1_2.c`, `gfxhub_v1_2.h`
- `gfxhub_v2_0.c`, `gfxhub_v2_0.h`
- `gfxhub_v2_1.c`, `gfxhub_v2_1.h`
- `gfxhub_v3_0.c`, `gfxhub_v3_0.h`
- `gfxhub_v3_0_3.c`, `gfxhub_v3_0_3.h`
- `gfxhub_v11_5_0.c`, `gfxhub_v11_5_0.h`
- `gfxhub_v12_0.c`, `gfxhub_v12_0.h`
- **Total**: ~14,000+ lines (estimated)

**MMHUB Files** (Multi-Media Hub):
- `mmhub_v1_0.c`, `mmhub_v1_0.h`
- `mmhub_v1_7.c`, `mmhub_v1_7.h`
- `mmhub_v1_8.c`, `mmhub_v1_8.h`
- `mmhub_v2_0.c`, `mmhub_v2_0.h`
- `mmhub_v2_3.c`, `mmhub_v2_3.h`
- `mmhub_v3_0.c`, `mmhub_v3_0.h`
- `mmhub_v3_0_1.c`, `mmhub_v3_0_1.h`
- `mmhub_v3_0_2.c`, `mmhub_v3_0_2.h`
- `mmhub_v3_3.c`, `mmhub_v3_3.h`
- `mmhub_v4_1_0.c`, `mmhub_v4_1_0.h`
- `mmhub_v9_4.c`, `mmhub_v9_4.h`
- **Total**: ~14,000+ lines (estimated)

**Purpose**: VMHUB files manage VM context registers and page table base addresses for different GPU hubs (GFXHUB for graphics, MMHUB for multimedia).

### SDMA Files (Page Table Update Engines)

**SDMA Version Files**:
- `sdma_v2_4.c`, `sdma_v3_0.c`, `sdma_v4_0.c`
- `sdma_v4_4_2.c`, `sdma_v5_0.c`, `sdma_v5_2.c`
- `sdma_v6_0.c`, `sdma_v7_0.c`
- `si_dma.c`, `cik_sdma.c`

**Key Functions** (from `sdma_v5_0.c`):
- `sdma_v5_0_vm_set_pte_pde()` - Update page tables using SDMA (line 1201)
- `sdma_v5_0_vm_update_page_table_base()` - Update page table base (line 1288)

---

## Complete File List

### Primary VM Files (27 files)

1. `amdgpu_vm.c` - 3,285 lines
2. `amdgpu_vm.h` - 710 lines
3. `amdgpu_vm_pt.c` - 970 lines
4. `amdgpu_vm_cpu.c` - 122 lines
5. `amdgpu_vm_sdma.c` - 296 lines
6. `amdgpu_vm_tlb_fence.c` - 111 lines
7. `amdgpu_gmc.c` - 1,692 lines
8. `amdgpu_gmc.h` - 471 lines
9. `gmc_v6_0.c` - 1,171 lines
10. `gmc_v6_0.h` - 29 lines
11. `gmc_v7_0.c` - 1,399 lines
12. `gmc_v7_0.h` - 30 lines
13. `gmc_v8_0.c` - 1,783 lines
14. `gmc_v8_0.h` - 31 lines
15. `gmc_v9_0.c` - 2,381 lines
16. `gmc_v9_0.h` - 31 lines
17. `gmc_v10_0.c` - 1,152 lines
18. `gmc_v10_0.h` - 30 lines
19. `gmc_v11_0.c` - 1,054 lines
20. `gmc_v11_0.h` - 30 lines
21. `gmc_v12_0.c` - 1,043 lines
22. `gmc_v12_0.h` - 30 lines
23. `amdgpu_object.c` - 1,735 lines
24. `amdgpu_object.h` - 374 lines
25. `amdgpu_ttm.c` - 3,278 lines
26. `amdgpu_ttm.h` - 275 lines
27. `amdgpu_amdkfd_gpuvm.c` - 3,674 lines

**Total Primary Files**: **27 files, 27,187 lines**

---

## Code Distribution

### By Functionality

| Functionality | Lines of Code | Percentage |
|---------------|---------------|------------|
| Core VM Management | 2,345 | 12.9% |
| Page Table Operations | 491 | 2.7% |
| Update Mechanisms | 217 | 1.2% |
| TLB Management | 64 | 0.4% |
| GMC Core | 1,429 | 7.9% |
| GMC Version-Specific | 7,483 | 41.1% |
| Buffer Objects | 1,242 | 6.8% |
| TTM Integration | 2,399 | 13.2% |
| KFD Integration | 2,528 | 13.9% |
| **TOTAL CODE** | **18,198** | **100%** |

### Code vs Comments Ratio

- **Code**: 18,198 lines (66.9%)
- **Comments**: 5,506 lines (20.2%)
- **Blanks**: 3,483 lines (12.8%)

**Comment Ratio**: ~30% (comments + blanks relative to code)

---

## Key File Responsibilities

### Virtual Address Management
- **Primary**: `amdgpu_vm.c` (lines 1882-2025)
- **Data Structures**: `amdgpu_vm.h` (lines 335-455)
- **Mapping Structure**: `amdgpu_object.h` (lines 64-73)

### Buffer Object Tracking
- **Primary**: `amdgpu_object.c` (lines 1-1735)
- **Structures**: `amdgpu_object.h` (lines 76-139)
- **VM-BO Link**: `amdgpu_vm.h` (lines 198-215)

### Physical Memory Mapping
- **GART Mapping**: `amdgpu_vm.c` (lines 968-981)
- **Address Resolution**: `amdgpu_vm.c` (lines 1207-1245)
- **PTE Encoding**: `amdgpu_gmc.c` (lines 162-177)

### Page Table Manipulations
- **PTE Updates**: `amdgpu_vm_pt.c` (lines 791-942)
- **PDE Updates**: `amdgpu_vm_pt.c` (lines 623-644)
- **Tree Walking**: `amdgpu_vm_pt.c` (lines 156-335)
- **CPU Updates**: `amdgpu_vm_cpu.c` (lines 69-96)
- **SDMA Updates**: `amdgpu_vm_sdma.c` (lines 218-289)

### Page Fault Handling
- **Fault Handler**: `amdgpu_vm.c` (lines 2979-3097)
- **Interrupt Processing**: `gmc_v9_0.c` (lines 565-621)
- **Fault Filtering**: `amdgpu_gmc.c` (lines 420-470)

---

## File Relationships

### Include Dependencies

**amdgpu_vm.c includes**:
```c
#include "amdgpu_vm.h"      // VM structures
#include "amdgpu_gmc.h"      // GMC functions
#include "amdgpu_object.h"   // BO structures
#include "amdgpu_ttm.h"      // TTM structures
```

**amdgpu_vm_pt.c includes**:
```c
#include "amdgpu_vm.h"       // VM structures
#include "amdgpu_gmc.h"     // GMC functions
```

**amdgpu_gmc.c includes**:
```c
#include "amdgpu_gmc.h"     // GMC structures
#include "amdgpu_vm.h"      // VM structures
#include "gmc_v[6-12]_0.h"  // Version-specific headers
```

---

## Evidence Summary

All file paths are relative to `/workspace/drivers/gpu/drm/amd/amdgpu/`:

| File | Purpose | Key Evidence Lines |
|------|---------|-------------------|
| `amdgpu_vm.c` | Core VM management | 1882-1921 (mapping), 1126-1271 (updates), 2979-3097 (faults) |
| `amdgpu_vm.h` | VM structures | 335-455 (struct amdgpu_vm), 198-215 (vm_bo_base) |
| `amdgpu_vm_pt.c` | Page table ops | 791-942 (PTE updates), 156-335 (tree walking) |
| `amdgpu_vm_cpu.c` | CPU updates | 69-96 (direct writes) |
| `amdgpu_vm_sdma.c` | SDMA updates | 218-289 (SDMA commands) |
| `amdgpu_gmc.c` | GMC core | 162-177 (PTE encoding), 420-470 (fault filtering) |
| `gmc_v9_0.c` | Vega GMC | 565-621 (fault handling), 463-477 (interrupts) |
| `amdgpu_object.c` | BO management | Buffer object lifecycle |
| `amdgpu_object.h` | BO structures | 64-73 (mapping), 76-99 (bo_va) |

---

## Conclusion

The AMDGPU virtual memory management system consists of **27 primary files** totaling **27,187 lines**, with **18,198 lines of actual code**. The codebase is well-documented with approximately 30% comments and blank lines, indicating good code maintainability.

The largest components are:
1. **GMC Version-Specific** (41.1% of code) - Hardware-specific implementations
2. **KFD Integration** (13.9% of code) - Compute context support
3. **TTM Integration** (13.2% of code) - Memory management
4. **Core VM Management** (12.9% of code) - Main VM logic

This modular structure allows for:
- Clean separation of concerns
- Hardware-specific optimizations per GPU generation
- Multiple update mechanisms (CPU/SDMA)
- Comprehensive fault handling
