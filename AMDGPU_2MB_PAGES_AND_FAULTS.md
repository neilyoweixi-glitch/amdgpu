# AMDGPU 2MB Page Support and Page Fault Handling

## Yes, AMDGPU Supports 2MB Pages!

### Fragment-Based Large Page Support

AMDGPU uses a **fragment** mechanism to support variable-sized pages:

- **Base page size**: 4KB (4096 bytes)
- **Fragment encoding**: Page size = `1 << (12 + frag)` bytes
- **2MB pages**: Achieved with `frag = 9` → `1 << (12 + 9) = 1 << 21 = 2MB`

### Fragment Size Limits

From `amdgpu_vm_pt.c`:

```c
if (params->adev->asic_type < CHIP_VEGA10)
    max_frag = params->adev->vm_manager.fragment_size;
else
    max_frag = 31;  // Vega10+ supports up to frag=31
```

**Fragment sizes by ASIC generation:**

1. **Pre-Vega10 (GCN 1.0-4.0)**:
   - Limited by `vm_manager.fragment_size` (typically 9 bits = 2MB max)
   - Default fragment size configurable via `amdgpu_vm_fragment_size` module parameter

2. **Vega10+ (GCN 5.0, RDNA 1/2/3)**:
   - Maximum fragment = 31 bits
   - Theoretical max page size: `1 << (12 + 31) = 2^43 = 8TB` (hardware-limited)
   - **2MB pages fully supported** (`frag = 9`)

### How Fragments Work

From `amdgpu_vm_pte_fragment()`:

```c
/**
 * The MC L1 TLB supports variable sized pages, based on a fragment
 * field in the PTE. When this field is set to a non-zero value, page
 * granularity is increased from 4KB to (1 << (12 + frag)). The PTE
 * flags are considered valid for all PTEs within the fragment range
 * and corresponding mappings are assumed to be physically contiguous.
 *
 * The L1 TLB can store a single PTE for the whole fragment,
 * significantly increasing the space available for translation
 * caching. This leads to large improvements in throughput when the
 * TLB is under pressure.
 *
 * The L2 TLB distributes small and large fragments into two
 * asymmetric partitions. The large fragment cache is significantly
 * larger. Thus, we try to use large fragments wherever possible.
 * Userspace can support this by aligning virtual base address and
 * allocation size to the fragment size.
 *
 * Starting with Vega10 the fragment size only controls the L1. The L2
 * is now directly feed with small/huge/giant pages from the walker.
 */
```

### Fragment Encoding in PTE

The fragment size is encoded in the PTE flags:

```c
#define AMDGPU_PTE_FRAG(x)    ((x & 0x1fULL) << 7)
```

- Bits 7-11 encode the fragment size (5 bits, values 0-31)
- `frag = 9` → 2MB pages
- `frag = 0` → 4KB pages (default)

### Huge Page Support

The code explicitly mentions huge page support:

```c
/* No huge page support before GMC v9 */
if (adev->asic_type < CHIP_VEGA10 &&
    (flags & AMDGPU_PTE_VALID)) {
    if (cursor.level != AMDGPU_VM_PTB) {
        /* Must go to leaf level */
    }
}
```

**Vega10+ (GCN 5.0, RDNA)**: Full huge page support at any page table level
**Pre-Vega10**: Limited to leaf-level PTEs only

## Page Fault Handling

### Yes, AMDGPU Has Comprehensive Page Fault Handling!

### Page Fault Detection

Page faults are detected via **VMC (Virtual Memory Controller) interrupts**:

**Interrupt Sources** (`gmc_v9_0.c`):
- `SOC15_IH_CLIENTID_VMC` - VMC interrupt client
- `SOC15_IH_CLIENTID_VMC1` - VMC1 interrupt client  
- `VMC_1_0__SRCID__VM_FAULT` - VM fault interrupt source

**Fault Types Detected**:
- Range protection faults
- Dummy page protection faults
- PDE0 protection faults
- Valid protection faults
- Read protection faults
- Write protection faults
- Execute protection faults

### Page Fault Handler Function

**Main handler**: `amdgpu_vm_handle_fault()` in `amdgpu_vm.c`

```c
/**
 * amdgpu_vm_handle_fault - graceful handling of VM faults.
 * @adev: amdgpu device pointer
 * @pasid: PASID of the VM
 * @ts: Timestamp of the fault
 * @vmid: VMID, only used for GFX 9.4.3.
 * @node_id: Node_id received in IH cookie. Only applicable for GFX 9.4.3.
 * @addr: Address of the fault
 * @write_fault: true is write fault, false is read fault
 *
 * Try to gracefully handle a VM fault. Return true if the fault was handled and
 * shouldn't be reported any more.
 */
bool amdgpu_vm_handle_fault(struct amdgpu_device *adev, u32 pasid,
                            u32 vmid, u32 node_id, uint64_t addr, uint64_t ts,
                            bool write_fault)
```

### Fault Handling Flow

1. **Interrupt Reception** (`amdgpu_irq_handler`):
   - GPU hardware raises interrupt
   - Interrupt handler (`amdgpu_ih_process`) processes interrupt vector
   - VMC fault interrupt detected

2. **Fault Processing** (`gmc_v9_0_process_interrupt`):
   - Extract fault information from interrupt vector:
     - Fault address
     - PASID (Process Address Space ID)
     - VMID (Virtual Memory ID)
     - Read/Write fault type
     - Fault status register

3. **Fault Handling** (`amdgpu_vm_handle_fault`):
   - Look up VM by PASID
   - For compute contexts: Try SVM (Shared Virtual Memory) range restore
   - For graphics contexts: Update page table entry
   - Options:
     - Redirect to dummy page (if `vm_fault_stop == NEVER`)
     - Set invalid PTE to force no-retry fault
     - Let hardware retry silently

4. **Page Table Update**:
   - Update PTE at fault address
   - Update parent PDEs if needed
   - Flush TLB if necessary

### Fault Status Information

Fault status is cached for debugging:

```c
struct amdgpu_vm_fault_info {
    uint64_t    addr;      // Fault address
    uint32_t    status;    // Fault status register
    unsigned int vmhub;    // Which VMHUB (GFXHUB/MMHUB)
};
```

**Fault Status Register Fields** (`VM_L2_PROTECTION_FAULT_STATUS`):
- `MORE_FAULTS`: Additional faults pending
- `WALKER_ERROR`: Page table walker error
- `PERMISSION_FAULTS`: Permission violation
- `MAPPING_ERROR`: Invalid mapping
- `CID`: Client ID that caused fault
- `RW`: Read/Write indicator
- `FED`: Fault Error Detail

### Fault Filtering and Deduplication

The driver implements fault filtering to prevent fault floods:

**Fault Ring Buffer** (`amdgpu_gmc.c`):
- Ring buffer of recent faults (256 entries)
- Hash table for fast lookup
- Timestamp-based expiration (5ms timeout)
- Prevents processing duplicate faults

```c
bool amdgpu_gmc_filter_faults(struct amdgpu_device *adev,
                              struct amdgpu_ih_ring *ih, uint64_t addr,
                              uint16_t pasid, uint64_t timestamp)
```

### Fault Handling Modes

Controlled by `amdgpu_vm_fault_stop` module parameter:

- **`AMDGPU_VM_FAULT_STOP_NEVER` (0)**: 
  - Redirect faults to dummy page
  - Continue execution
  - Default mode

- **`AMDGPU_VM_FAULT_STOP_FIRST` (1)**:
  - Print first fault, then continue

- **`AMDGPU_VM_FAULT_STOP_ALWAYS` (2)**:
  - Stop on every fault
  - Useful for debugging

### Compute Context Fault Handling

For compute contexts (SVM), faults trigger page migration:

```c
if (is_compute_context && !svm_range_restore_pages(adev, pasid, vmid,
    node_id, addr, ts, write_fault)) {
    // SVM handles the fault by migrating pages
    return true;
}
```

### Key Files for Page Fault Handling

1. **`amdgpu_vm.c`**: `amdgpu_vm_handle_fault()` - Main fault handler
2. **`amdgpu_irq.c`**: Interrupt processing infrastructure
3. **`gmc_v9_0.c`**: VMC interrupt handler for Vega10+
4. **`amdgpu_gmc.c`**: Fault filtering and caching
5. **`amdgpu_ih.c`**: Interrupt vector processing

### Summary

✅ **2MB Pages**: Fully supported via fragment mechanism (`frag = 9`)
- Vega10+ supports up to `frag = 31` (theoretical 8TB pages)
- Hardware automatically uses large pages when possible
- Userspace can optimize by aligning allocations

✅ **Page Fault Handling**: Comprehensive fault handling system
- Hardware interrupts for VM faults
- Graceful fault recovery
- Fault deduplication and filtering
- Support for both graphics and compute contexts
- Debugging support with fault status caching
