# AMDGPU Virtual Memory Management - Comprehensive Code Analysis

## Table of Contents
1. [Virtual Address Management](#1-virtual-address-management)
2. [Buffer Object (BO) Management](#2-buffer-object-bo-management)
3. [Physical Memory Mapping](#3-physical-memory-mapping)
4. [Page Table Manipulations](#4-page-table-manipulations)

---

## 1. Virtual Address Management

### 1.1 Virtual Address Space Structure

**Evidence**: `amdgpu_vm.h:335-341`

```c
struct amdgpu_vm {
    /* tree of virtual addresses mapped */
    struct rb_root_cached va;  // Interval tree for VA ranges
    ...
}
```

**Key Components**:
- **Interval Tree**: Uses `rb_root_cached` to efficiently track VA ranges
- **Mapping Structure**: `amdgpu_bo_va_mapping` represents each VA range

**Evidence**: `amdgpu_object.h:64-73`

```c
struct amdgpu_bo_va_mapping {
    struct amdgpu_bo_va      *bo_va;
    struct list_head         list;
    struct rb_node           rb;          // For interval tree
    uint64_t                 start;       // Start VA (in GPU pages)
    uint64_t                 last;        // End VA (in GPU pages)
    uint64_t                 __subtree_last;  // For interval tree
    uint64_t                 offset;      // Offset into BO
    uint32_t                 flags;      // Page flags
};
```

### 1.2 Virtual Address Range Reservation

**Reserved VA Regions** (`amdgpu_vm.h:167-181`):

```c
/* Reserve space at top/bottom of address space for kernel use */
#define AMDGPU_VA_RESERVED_CSA_SIZE        (2ULL << 20)      // 2MB
#define AMDGPU_VA_RESERVED_SEQ64_SIZE      (2ULL << 20)      // 2MB
#define AMDGPU_VA_RESERVED_TRAP_SIZE       (2ULL << 12)       // 8KB
#define AMDGPU_VA_RESERVED_BOTTOM          (1ULL << 16)       // 64KB
```

**Evidence**: Lines 168-181 in `amdgpu_vm.h` show reserved regions at:
- **Top of address space**: CSA (2MB) + SEQ64 (2MB) + TRAP (8KB)
- **Bottom**: 64KB reserved

### 1.3 Virtual Address Mapping Operations

#### 1.3.1 Adding a Mapping

**Function**: `amdgpu_vm_bo_map()`  
**Location**: `amdgpu_vm.c:1882-1921`

**Evidence**:
```c
int amdgpu_vm_bo_map(struct amdgpu_device *adev,
                     struct amdgpu_bo_va *bo_va,
                     uint64_t saddr, uint64_t offset,
                     uint64_t size, uint32_t flags)
{
    // Convert byte addresses to GPU page numbers
    saddr /= AMDGPU_GPU_PAGE_SIZE;
    eaddr = saddr + (size - 1) / AMDGPU_GPU_PAGE_SIZE;
    
    // Check for conflicts using interval tree
    tmp = amdgpu_vm_it_iter_first(&vm->va, saddr, eaddr);
    if (tmp) {
        // Conflict detected
        return -EINVAL;
    }
    
    // Create mapping structure
    mapping->start = saddr;
    mapping->last = eaddr;
    mapping->offset = offset;
    mapping->flags = flags;
    
    // Insert into interval tree
    amdgpu_vm_bo_insert_map(adev, bo_va, mapping);
}
```

**Key Operations**:
1. **Address Validation**: `amdgpu_vm_verify_parameters()` (lines 1835-1863)
   - Checks alignment to GPU page size
   - Validates address range doesn't exceed `max_pfn`
   - Ensures offset + size fits within BO

2. **Conflict Detection**: Uses interval tree to detect overlapping mappings
   - **Evidence**: Line 1900 - `amdgpu_vm_it_iter_first()` searches for overlaps

3. **Mapping Insertion**: `amdgpu_vm_bo_insert_map()` (lines 1814-1832)
   - Adds to `bo_va->invalids` list
   - Inserts into VM's interval tree
   - Handles PRT (Partial Resident Texture) flags

#### 1.3.2 Removing a Mapping

**Function**: `amdgpu_vm_bo_unmap()`  
**Location**: `amdgpu_vm.c:1993-2025`

**Evidence**:
```c
int amdgpu_vm_bo_unmap(struct amdgpu_device *adev,
                       struct amdgpu_bo_va *bo_va,
                       uint64_t saddr)
{
    saddr /= AMDGPU_GPU_PAGE_SIZE;
    
    // Search in valids list first
    list_for_each_entry(mapping, &bo_va->valids, list) {
        if (mapping->start == saddr)
            break;
    }
    
    // If not found, search invalids
    if (&mapping->list == &bo_va->valids) {
        list_for_each_entry(mapping, &bo_va->invalids, list) {
            if (mapping->start == saddr)
                break;
        }
    }
    
    // Remove from interval tree and lists
    amdgpu_vm_it_remove(mapping, &vm->va);
    list_del(&mapping->list);
    
    // Move to freed list for later PT update
    list_add(&mapping->list, &vm->freed);
}
```

### 1.4 Virtual Address Lookup

**Function**: `amdgpu_vm_bo_lookup_mapping()`  
**Location**: `amdgpu_vm.c:584` (declared in header)

**Evidence**: Uses interval tree search:
```c
struct amdgpu_bo_va_mapping *amdgpu_vm_bo_lookup_mapping(struct amdgpu_vm *vm,
                                                         uint64_t addr)
```

**Interval Tree Implementation**: `amdgpu_vm.c:90-97`
```c
#define START(node) ((node)->start)
#define LAST(node) ((node)->last)

INTERVAL_TREE_DEFINE(struct amdgpu_bo_va_mapping, rb, uint64_t, __subtree_last,
                     START, LAST, static, amdgpu_vm_it)
```

---

## 2. Buffer Object (BO) Management

### 2.1 Buffer Object Structure

**Evidence**: `amdgpu_object.h:101-139`

```c
struct amdgpu_bo {
    u32                     preferred_domains;
    u32                     allowed_domains;
    struct ttm_place        placements[AMDGPU_BO_MAX_PLACEMENTS];
    struct ttm_placement    placement;
    struct ttm_buffer_object tbo;          // TTM buffer object
    struct ttm_bo_kmap_obj  kmap;
    u64                     flags;
    struct amdgpu_vm_bo_base *vm_bo;      // Per-VM structure
    struct amdgpu_bo        *parent;      // For page tables
    ...
};
```

### 2.2 BO-VM Association

**Structure**: `amdgpu_vm_bo_base`  
**Location**: `amdgpu_vm.h:198-215`

**Evidence**:
```c
struct amdgpu_vm_bo_base {
    struct amdgpu_vm        *vm;          // Associated VM
    struct amdgpu_bo         *bo;          // Associated BO
    struct amdgpu_vm_bo_base *next;       // Linked list
    struct list_head         vm_status;    // State machine list
    bool                     shared;       // Shared across VMs
    bool                     moved;        // BO moved, needs update
};
```

**State Machine Lists** (`amdgpu_vm.h:363-399`):
- `evicted`: BOs needing validation
- `relocated`: Page tables that relocated
- `moved`: Per-VM BOs that moved
- `idle`: BOs not in state machine
- `evicted_user`: User mode queue BOs
- `invalidated`: BOs needing PT update
- `done`: BOs updated in PTs
- `freed`: Mappings freed but not updated in PTs

### 2.3 BO Virtual Address Tracking

**Structure**: `amdgpu_bo_va`  
**Location**: `amdgpu_object.h:76-99`

**Evidence**:
```c
struct amdgpu_bo_va {
    struct amdgpu_vm_bo_base base;        // Base VM-BO link
    
    unsigned                 ref_count;   // Reference count
    struct dma_fence         *last_pt_update;  // Last PT update fence
    
    struct list_head         invalids;     // Invalid mappings
    struct list_head         valids;       // Valid mappings
    
    bool                     cleared;      // Mappings cleared
    bool                     is_xgmi;      // XGMI accessible
    unsigned int             queue_refcount;  // Queue references
};
```

**Key Operations**:

1. **Adding BO to VM**: `amdgpu_vm_bo_add()`  
   **Location**: `amdgpu_vm.c:1774-1802`

   **Evidence**:
   ```c
   struct amdgpu_bo_va *amdgpu_vm_bo_add(struct amdgpu_device *adev,
                                         struct amdgpu_vm *vm,
                                         struct amdgpu_bo *bo)
   {
       bo_va = kzalloc(sizeof(struct amdgpu_bo_va), GFP_KERNEL);
       amdgpu_vm_bo_base_init(&bo_va->base, vm, bo);
       bo_va->ref_count = 1;
       bo_va->last_pt_update = dma_fence_get_stub();
       INIT_LIST_HEAD(&bo_va->valids);
       INIT_LIST_HEAD(&bo_va->invalids);
       ...
   }
   ```

2. **Finding BO in VM**: `amdgpu_vm_bo_find()`  
   **Location**: `amdgpu_vm.c:938-954`

   **Evidence**:
   ```c
   struct amdgpu_bo_va *amdgpu_vm_bo_find(struct amdgpu_vm *vm,
                                          struct amdgpu_bo *bo)
   {
       struct amdgpu_vm_bo_base *base;
       
       for (base = bo->vm_bo; base; base = base->next) {
           if (base->vm == vm)
               return container_of(base, struct amdgpu_bo_va, base);
       }
       return NULL;
   }
   ```

### 2.4 BO Memory Domains

**Memory Type to Domain Mapping**: `amdgpu_object.h:165-192`

**Evidence**:
```c
static inline unsigned amdgpu_mem_type_to_domain(u32 mem_type)
{
    switch (mem_type) {
    case TTM_PL_VRAM:
        return AMDGPU_GEM_DOMAIN_VRAM;
    case TTM_PL_TT:
        return AMDGPU_GEM_DOMAIN_GTT;
    case TTM_PL_SYSTEM:
        return AMDGPU_GEM_DOMAIN_CPU;
    case AMDGPU_PL_GDS:
        return AMDGPU_GEM_DOMAIN_GDS;
    ...
    }
}
```

**Supported Domains**:
- **VRAM**: Video RAM (on-GPU memory)
- **GTT**: Graphics Translation Table (system memory via GART)
- **CPU**: System memory
- **GDS/GWS/OA**: Special GPU memory types
- **DOORBELL**: Doorbell memory
- **DGMA**: Direct GMA memory

---

## 3. Physical Memory Mapping

### 3.1 Physical Address Resolution

**Function**: `amdgpu_vm_map_gart()`  
**Location**: `amdgpu_vm.c:968-981`

**Evidence**:
```c
uint64_t amdgpu_vm_map_gart(const dma_addr_t *pages_addr, uint64_t addr)
{
    uint64_t result;
    
    /* page table offset */
    result = pages_addr[addr >> PAGE_SHIFT];
    
    /* in case cpu page size != gpu page size*/
    result |= addr & (~PAGE_MASK);
    
    result &= 0xFFFFFFFFFFFFF000ULL;  // 48-bit address, 4KB aligned
    
    return result;
}
```

**Purpose**: Converts virtual address to physical address using DMA address array

### 3.2 Physical Address Sources

**Evidence**: `amdgpu_vm.c:1207-1245` in `amdgpu_vm_update_range()`

```c
if (pages_addr) {
    // System memory: use DMA addresses
    if (contiguous) {
        addr = pages_addr[cursor.start >> PAGE_SHIFT];
        params.pages_addr = NULL;  // Direct mapping
    } else {
        addr = cursor.start;
        params.pages_addr = pages_addr;  // Scattered pages
    }
} else if (flags & (AMDGPU_PTE_VALID | AMDGPU_PTE_PRT_FLAG(adev))) {
    // VRAM: use vram_base + offset
    addr = vram_base + cursor.start;
} else {
    // Invalid/unmapped
    addr = 0;
}
```

**Physical Address Types**:

1. **VRAM Addresses**:
   - **Evidence**: Line 1241 - `addr = vram_base + cursor.start`
   - Uses `vram_base_offset` from VM manager
   - Direct GPU memory access

2. **System Memory (GTT)**:
   - **Evidence**: Lines 1208-1239
   - Uses `pages_addr` array (DMA addresses)
   - Can be contiguous or scattered
   - Contiguous check: `pages_addr[pfn + 1] == pages_addr[pfn] + PAGE_SIZE`

3. **DGMA (Direct GMA)**:
   - **Evidence**: Lines 1194-1206
   - Special memory type for direct GPU memory access
   - Uses `dgma_addr` or `dgma_import_base`

### 3.3 Physical Address Encoding in PTE

**Function**: `amdgpu_gmc_set_pte_pde()`  
**Location**: `amdgpu_gmc.c:162-177`

**Evidence**:
```c
int amdgpu_gmc_set_pte_pde(struct amdgpu_device *adev, void *cpu_pt_addr,
                            uint32_t gpu_page_idx, uint64_t addr,
                            uint64_t flags)
{
    void __iomem *ptr = (void *)cpu_pt_addr;
    uint64_t value;
    
    // Physical address: lower 48 bits, 4KB aligned
    value = addr & 0x0000FFFFFFFFF000ULL;
    value |= flags;  // OR in flags
    
    // Write 64-bit PTE entry
    writeq(value, ptr + (gpu_page_idx * 8));
    
    return 0;
}
```

**PTE Format**:
- **Bits 0-47**: Physical address (48-bit, 4KB aligned)
- **Bits 48-63**: Flags (valid, permissions, memory type, etc.)

### 3.4 GPU Offset Calculation

**Function**: `amdgpu_bo_gpu_offset()`  
**Location**: `amdgpu_object.h:324` (declared)

**Purpose**: Get GPU-visible address of a BO

**Evidence**: Used throughout codebase:
- Line 121 in `amdgpu_gmc.c`: `*addr = amdgpu_bo_gpu_offset(bo);`
- Used for VRAM-backed BOs

---

## 4. Page Table Manipulations

### 4.1 Page Table Structure

**Page Table Levels**: `amdgpu_vm.h:187-195`

**Evidence**:
```c
enum amdgpu_vm_level {
    AMDGPU_VM_PDB2,  // Page Directory Block 2 (top)
    AMDGPU_VM_PDB1,  // Page Directory Block 1
    AMDGPU_VM_PDB0,  // Page Directory Block 0
    AMDGPU_VM_PTB    // Page Table Block (leaf)
};
```

**Page Table Entry Structure**: `amdgpu_vm_pt.c:33-38`

**Evidence**:
```c
struct amdgpu_vm_pt_cursor {
    uint64_t                pfn;          // Page frame number
    struct amdgpu_vm_bo_base *parent;     // Parent directory
    struct amdgpu_vm_bo_base *entry;      // Current entry
    unsigned int            level;        // Current level
};
```

### 4.2 Page Table Allocation

**Function**: `amdgpu_vm_pt_create()`  
**Location**: `amdgpu_vm_pt.c:438-477`

**Evidence**:
```c
int amdgpu_vm_pt_create(struct amdgpu_device *adev, struct amdgpu_vm *vm,
                        int level, bool immediate, struct amdgpu_bo_vm **vmbo,
                        int32_t xcp_id)
{
    bp.size = amdgpu_vm_pt_size(adev, level);
    bp.byte_align = AMDGPU_GPU_PAGE_SIZE;
    
    if (!adev->gmc.is_app_apu)
        bp.domain = AMDGPU_GEM_DOMAIN_VRAM;  // VRAM for dGPUs
    else
        bp.domain = AMDGPU_GEM_DOMAIN_GTT;  // GTT for APUs
    
    bp.flags = AMDGPU_GEM_CREATE_VRAM_CONTIGUOUS |
               AMDGPU_GEM_CREATE_CPU_GTT_USWC;
    
    num_entries = amdgpu_vm_pt_num_entries(adev, level);
    bp.bo_ptr_size = struct_size((*vmbo), entries, num_entries);
    
    return amdgpu_bo_create_vm(adev, &bp, vmbo);
}
```

**Page Table Sizes** (`amdgpu_vm_pt.c:74-90`):
- **Root level**: Based on `max_pfn` and level shift
- **Intermediate levels**: 512 entries (9 bits)
- **Leaf level (PTB)**: `AMDGPU_VM_PTE_COUNT(adev)` entries
- **Entry size**: 8 bytes (64-bit)

### 4.3 Page Table Tree Walking

**Tree Walking Functions**: `amdgpu_vm_pt.c:156-335`

**Evidence**:

1. **Start Walk**: `amdgpu_vm_pt_start()` (lines 156-164)
   ```c
   static void amdgpu_vm_pt_start(struct amdgpu_device *adev,
                                   struct amdgpu_vm *vm, uint64_t start,
                                   struct amdgpu_vm_pt_cursor *cursor)
   {
       cursor->pfn = start;
       cursor->parent = NULL;
       cursor->entry = &vm->root;
       cursor->level = adev->vm_manager.root_level;
   }
   ```

2. **Descend to Child**: `amdgpu_vm_pt_descendant()` (lines 176-193)
   ```c
   static bool amdgpu_vm_pt_descendant(struct amdgpu_device *adev,
                                       struct amdgpu_vm_pt_cursor *cursor)
   {
       if ((cursor->level == AMDGPU_VM_PTB) || !cursor->entry ||
           !cursor->entry->bo)
           return false;
       
       mask = amdgpu_vm_pt_entries_mask(adev, cursor->level);
       shift = amdgpu_vm_pt_level_shift(adev, cursor->level);
       
       ++cursor->level;
       idx = (cursor->pfn >> shift) & mask;
       cursor->parent = cursor->entry;
       cursor->entry = &to_amdgpu_bo_vm(cursor->entry->bo)->entries[idx];
       return true;
   }
   ```

3. **Move to Sibling**: `amdgpu_vm_pt_sibling()` (lines 205-228)
4. **Move to Parent**: `amdgpu_vm_pt_ancestor()` (lines 239-248)
5. **Next Entry**: `amdgpu_vm_pt_next()` (lines 258-273)

### 4.4 Page Table Entry Updates

#### 4.4.1 PTE Update Function

**Function**: `amdgpu_vm_ptes_update()`  
**Location**: `amdgpu_vm_pt.c:791-942`

**Evidence**:
```c
int amdgpu_vm_ptes_update(struct amdgpu_vm_update_params *params,
                          uint64_t start, uint64_t end,
                          uint64_t dst, uint64_t flags)
{
    // Calculate fragment size
    amdgpu_vm_pte_fragment(params, frag_start, end, flags, &frag, &frag_end);
    
    // Start tree walk
    amdgpu_vm_pt_start(adev, params->vm, start, &cursor);
    
    while (cursor.pfn < end) {
        // Allocate page tables if needed
        if (!params->unlocked) {
            r = amdgpu_vm_pt_alloc(params->adev, params->vm,
                                   &cursor, params->immediate);
        }
        
        // Calculate entry position
        shift = amdgpu_vm_pt_level_shift(adev, cursor.level);
        mask = amdgpu_vm_pt_entries_mask(adev, cursor.level);
        pe_start = ((cursor.pfn >> shift) & mask) * 8;
        
        // Update PTE entries
        amdgpu_vm_pte_update_flags(params, to_amdgpu_bo_vm(pt),
                                   cursor.level, pe_start, dst,
                                   nptes, incr, upd_flags);
        
        // Move to next entry
        amdgpu_vm_pt_next(adev, &cursor);
    }
}
```

**Key Steps**:
1. **Fragment Calculation**: Determines optimal page size
2. **Tree Walk**: Traverses page table hierarchy
3. **Entry Calculation**: Computes PTE offset within page table
4. **Update**: Writes PTE entries via update function

#### 4.4.2 PTE Update Implementation

**Function**: `amdgpu_vm_pte_update_flags()`  
**Location**: `amdgpu_vm_pt.c:672-714`

**Evidence**:
```c
static void amdgpu_vm_pte_update_flags(struct amdgpu_vm_update_params *params,
                                       struct amdgpu_bo_vm *pt,
                                       unsigned int level,
                                       uint64_t pe, uint64_t addr,
                                       unsigned int count, uint32_t incr,
                                       uint64_t flags)
{
    struct amdgpu_device *adev = params->adev;
    
    if (level != AMDGPU_VM_PTB) {
        // PDE: handle as PTE for Vega10+
        flags |= AMDGPU_PDE_PTE_FLAG(params->adev);
        amdgpu_gmc_get_vm_pde(adev, level, &addr, &flags);
    } else if (adev->asic_type >= CHIP_VEGA10 &&
               !(flags & AMDGPU_PTE_VALID) &&
               !(flags & AMDGPU_PTE_PRT_FLAG(params->adev))) {
        // Workaround for fault priority problem on GMC9
        flags |= AMDGPU_PTE_EXECUTABLE;
    }
    
    // Update no-retry flags for TF
    if (level == AMDGPU_VM_PTB)
        amdgpu_vm_pte_update_noretry_flags(adev, &flags);
    
    // NUMA-aware MTYPE override for APUs
    if ((flags & AMDGPU_PTE_SYSTEM) && (adev->flags & AMD_IS_APU) &&
        adev->gmc.gmc_funcs->override_vm_pte_flags &&
        num_possible_nodes() > 1 && !params->pages_addr && params->allow_override)
        amdgpu_gmc_override_vm_pte_flags(adev, params->vm, addr, &flags);
    
    // Call update function (CPU or SDMA)
    params->vm->update_funcs->update(params, pt, pe, addr, count, incr, flags);
}
```

#### 4.4.3 CPU-Based PTE Updates

**Function**: `amdgpu_vm_cpu_update()`  
**Location**: `amdgpu_vm_cpu.c:69-96`

**Evidence**:
```c
static int amdgpu_vm_cpu_update(struct amdgpu_vm_update_params *p,
                                struct amdgpu_bo_vm *vmbo, uint64_t pe,
                                uint64_t addr, unsigned count, uint32_t incr,
                                uint64_t flags)
{
    unsigned int i;
    uint64_t value;
    long r;
    
    // Wait for BO to be ready
    r = dma_resv_wait_timeout(amdkcl_ttm_resvp(&vmbo->bo.tbo), 
                              DMA_RESV_USAGE_KERNEL,
                              true, MAX_SCHEDULE_TIMEOUT);
    
    // Get CPU virtual address
    pe += (unsigned long)amdgpu_bo_kptr(&vmbo->bo);
    
    // Write each PTE entry
    for (i = 0; i < count; i++) {
        value = p->pages_addr ?
            amdgpu_vm_map_gart(p->pages_addr, addr) :  // System memory
            addr;                                      // VRAM
        
        amdgpu_gmc_set_pte_pde(p->adev, (void *)(uintptr_t)pe,
                               i, value, flags);
        addr += incr;
    }
    return 0;
}
```

#### 4.4.4 SDMA-Based PTE Updates

**Function**: `amdgpu_vm_sdma_update()`  
**Location**: `amdgpu_vm_sdma.c:218-289`

**Evidence**:
```c
static int amdgpu_vm_sdma_update(struct amdgpu_vm_update_params *p,
                                 struct amdgpu_bo_vm *vmbo, uint64_t pe,
                                 uint64_t addr, unsigned count, uint32_t incr,
                                 uint64_t flags)
{
    struct amdgpu_bo *bo = &vmbo->bo;
    struct amdgpu_ib *ib = p->job->ibs;
    
    // Wait for PD/PT moves
    dma_resv_iter_begin(&cursor, amdkcl_ttm_resvp(&bo->tbo), DMA_RESV_USAGE_KERNEL);
    dma_resv_for_each_fence_unlocked(&cursor, fence) {
        drm_sched_job_add_dependency(&p->job->base, fence);
    }
    
    if (!p->pages_addr) {
        // Direct mapping: use set_pte_pde command
        amdgpu_vm_sdma_set_ptes(p, bo, pe, addr, count, incr, flags);
    } else {
        // Scattered pages: copy PTEs from IB
        pte = (uint64_t *)&(p->job->ibs->ptr[p->num_dw_left]);
        for (i = 0; i < nptes; ++i, addr += incr) {
            pte[i] = amdgpu_vm_map_gart(p->pages_addr, addr);
            pte[i] |= flags;
        }
        amdgpu_vm_sdma_copy_ptes(p, bo, pe, nptes);
    }
}
```

### 4.5 PDE (Page Directory Entry) Updates

**Function**: `amdgpu_vm_pde_update()`  
**Location**: `amdgpu_vm_pt.c:623-644`

**Evidence**:
```c
int amdgpu_vm_pde_update(struct amdgpu_vm_update_params *params,
                         struct amdgpu_vm_bo_base *entry)
{
    struct amdgpu_vm_bo_base *parent = amdgpu_vm_pt_parent(entry);
    struct amdgpu_bo *bo, *pbo;
    struct amdgpu_vm *vm = params->vm;
    uint64_t pde, pt, flags;
    unsigned int level;
    
    // Calculate level
    bo = parent->bo;
    for (level = 0, pbo = bo->parent; pbo; ++level)
        pbo = pbo->parent;
    level += params->adev->vm_manager.root_level;
    
    // Get PDE address and flags
    amdgpu_gmc_get_pde_for_bo(entry->bo, level, &pt, &flags);
    
    // Calculate PDE offset in parent
    pde = (entry - to_amdgpu_bo_vm(parent->bo)->entries) * 8;
    
    // Update PDE
    return vm->update_funcs->update(params, to_amdgpu_bo_vm(bo), 
                                    pde, pt, 1, 0, flags);
}
```

**PDE Format**: Points to child page table/directory
- **Address**: Physical address of child PT/PD
- **Flags**: Page table attributes

### 4.6 Page Table Range Updates

**Function**: `amdgpu_vm_update_range()`  
**Location**: `amdgpu_vm.c:1126-1271`

**Evidence**:
```c
int amdgpu_vm_update_range(struct amdgpu_device *adev, struct amdgpu_vm *vm,
                          bool immediate, bool unlocked, bool flush_tlb,
                          bool allow_override, struct amdgpu_sync *sync,
                          uint64_t start, uint64_t last, uint64_t flags,
                          uint64_t offset, uint64_t vram_base,
                          struct ttm_resource *res, dma_addr_t *pages_addr,
                          struct dma_fence **fence)
{
    // Initialize update parameters
    params.adev = adev;
    params.vm = vm;
    params.immediate = immediate;
    params.pages_addr = pages_addr;
    
    // Prepare update (allocate job, sync fences)
    r = vm->update_funcs->prepare(&params, sync);
    
    // Iterate over resource chunks
    amdgpu_res_first(pages_addr ? NULL : res, offset,
                     (last - start + 1) * AMDGPU_GPU_PAGE_SIZE, &cursor);
    while (cursor.remaining) {
        // Calculate physical address
        if (pages_addr) {
            // System memory
            addr = pages_addr[cursor.start >> PAGE_SHIFT];
        } else if (flags & AMDGPU_PTE_VALID) {
            // VRAM
            addr = vram_base + cursor.start;
        }
        
        // Update PTEs for this chunk
        r = amdgpu_vm_ptes_update(&params, start, tmp, addr, flags);
        
        // Move to next chunk
        amdgpu_res_next(&cursor, num_entries * AMDGPU_GPU_PAGE_SIZE);
    }
    
    // Commit update (submit job, get fence)
    r = vm->update_funcs->commit(&params, fence);
    
    // Handle TLB flush
    if (params.needs_flush) {
        amdgpu_vm_tlb_flush(&params, fence, tlb_cb);
    }
}
```

**Key Features**:
1. **Resource Cursor**: Iterates over memory resource chunks
2. **Contiguous Detection**: Optimizes for contiguous pages
3. **Fragment Handling**: Uses largest possible page size
4. **TLB Flush**: Invalidates TLB after updates

### 4.7 Page Table Clearing

**Function**: `amdgpu_vm_pt_clear()`  
**Location**: `amdgpu_vm_pt.c:359-426`

**Evidence**:
```c
int amdgpu_vm_pt_clear(struct amdgpu_device *adev, struct amdgpu_vm *vm,
                      struct amdgpu_bo_vm *vmbo, bool immediate)
{
    // Determine level in hierarchy
    if (ancestor->parent) {
        ++level;
        while (ancestor->parent->parent) {
            ++level;
            ancestor = ancestor->parent;
        }
    }
    
    entries = amdgpu_bo_size(bo) / 8;
    
    // Map table for CPU access
    r = vm->update_funcs->map_table(vmbo);
    
    // Prepare update
    r = vm->update_funcs->prepare(&params, NULL);
    
    // Clear all entries (write zeros)
    value = 0;
    flags = 0;
    if (adev->asic_type >= CHIP_VEGA10) {
        if (level != AMDGPU_VM_PTB) {
            flags |= AMDGPU_PDE_PTE_FLAG(adev);
            amdgpu_gmc_get_vm_pde(adev, level, &value, &flags);
        } else {
            flags = AMDGPU_PTE_EXECUTABLE;  // Workaround
        }
    }
    
    // Update all entries to zero/scratch
    r = vm->update_funcs->update(&params, vmbo, addr, 0, entries,
                                 value, flags);
    
    // Commit
    r = vm->update_funcs->commit(&params, NULL);
}
```

### 4.8 Page Table Freeing

**Function**: `amdgpu_vm_pt_free()`  
**Location**: `amdgpu_vm_pt.c:535-546`

**Evidence**:
```c
static void amdgpu_vm_pt_free(struct amdgpu_vm_bo_base *entry)
{
    if (!entry->bo)
        return;
    
    // Update memory statistics
    amdgpu_vm_update_stats(entry, entry->bo->tbo.resource, -1);
    
    // Clear VM-BO link
    entry->bo->vm_bo = NULL;
    ttm_bo_set_bulk_move(&entry->bo->tbo, NULL);
    
    // Remove from VM lists
    list_del(&entry->vm_status);
    
    // Release BO reference
    amdgpu_bo_unref(&entry->bo);
}
```

**Free Root**: `amdgpu_vm_pt_free_root()` (lines 604-613)
- Recursively frees entire page table tree
- Uses DFS traversal

---

## Summary of Key Data Structures

### Virtual Address → Physical Address Flow

```
Virtual Address (GPU VA)
    ↓
amdgpu_bo_va_mapping (interval tree)
    ↓
amdgpu_bo_va → amdgpu_bo
    ↓
ttm_resource (physical memory)
    ↓
Physical Address (VRAM offset or DMA address)
    ↓
PTE Entry (64-bit: 48-bit PA + 16-bit flags)
```

### Page Table Hierarchy

```
Root PD (amdgpu_vm.root.bo)
    ↓
PDB2/PDB1/PDB0 (intermediate directories)
    ↓
PTB (leaf page tables)
    ↓
PTE entries (pointing to physical pages)
```

### Update Mechanisms

1. **CPU Updates**: Direct memory writes (`amdgpu_vm_cpu_funcs`)
2. **SDMA Updates**: GPU DMA engine (`amdgpu_vm_sdma_funcs`)
3. **Selection**: Based on `vm->use_cpu_for_update` flag

---

## Code References Summary

| Component | Key Files | Key Functions |
|-----------|-----------|---------------|
| **Virtual Addresses** | `amdgpu_vm.c`, `amdgpu_vm.h` | `amdgpu_vm_bo_map()`, `amdgpu_vm_bo_unmap()`, `amdgpu_vm_bo_lookup_mapping()` |
| **Buffer Objects** | `amdgpu_object.h`, `amdgpu_vm.c` | `amdgpu_vm_bo_add()`, `amdgpu_vm_bo_find()`, `amdgpu_bo_create()` |
| **Physical Memory** | `amdgpu_vm.c`, `amdgpu_gmc.c` | `amdgpu_vm_map_gart()`, `amdgpu_bo_gpu_offset()`, `amdgpu_gmc_set_pte_pde()` |
| **Page Tables** | `amdgpu_vm_pt.c`, `amdgpu_vm_cpu.c`, `amdgpu_vm_sdma.c` | `amdgpu_vm_ptes_update()`, `amdgpu_vm_pde_update()`, `amdgpu_vm_pt_create()` |

---

## Evidence Locations

- **Virtual Address Management**: `amdgpu_vm.c:1882-2025`, `amdgpu_vm.h:335-455`
- **Buffer Object Tracking**: `amdgpu_object.h:64-139`, `amdgpu_vm.c:1774-1802`
- **Physical Address Resolution**: `amdgpu_vm.c:968-981`, `amdgpu_vm.c:1207-1245`
- **Page Table Operations**: `amdgpu_vm_pt.c:791-942`, `amdgpu_vm_pt.c:359-426`
- **PTE Updates**: `amdgpu_vm_cpu.c:69-96`, `amdgpu_vm_sdma.c:218-289`
- **PDE Updates**: `amdgpu_vm_pt.c:623-644`
