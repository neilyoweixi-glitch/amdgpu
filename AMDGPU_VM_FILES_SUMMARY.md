# AMDGPU Virtual Memory Management - File Summary

## Quick Reference

### Total Statistics
- **Total Files**: 27 files
- **Total Lines**: 27,187 lines
- **Code Lines**: ~18,198 lines (66.9%)
- **Comment Lines**: ~5,506 lines (20.2%)
- **Blank Lines**: ~3,483 lines (12.8%)

---

## File List with Line Counts

### Core VM Management (3,995 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_vm.c` | 3,285 | C | Main VM implementation |
| `amdgpu_vm.h` | 710 | H | VM structures & declarations |

### Page Table Operations (970 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_vm_pt.c` | 970 | C | Page table tree walking & manipulation |

### Update Mechanisms (418 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_vm_cpu.c` | 122 | C | CPU-based page table updates |
| `amdgpu_vm_sdma.c` | 296 | C | SDMA-based page table updates |

### TLB Management (111 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_vm_tlb_fence.c` | 111 | C | TLB flush fence management |

### GMC Core (2,163 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_gmc.c` | 1,692 | C | Graphics Memory Controller core |
| `amdgpu_gmc.h` | 471 | H | GMC structures & function pointers |

### GMC Version-Specific (10,194 lines)

| File | Lines | Type | GPU Generation |
|------|-------|------|----------------|
| `gmc_v6_0.c` | 1,171 | C | GFX6 (Southern Islands) |
| `gmc_v6_0.h` | 29 | H | |
| `gmc_v7_0.c` | 1,399 | C | GFX7 (Sea Islands) |
| `gmc_v7_0.h` | 30 | H | |
| `gmc_v8_0.c` | 1,783 | C | GFX8 (Volcanic Islands) |
| `gmc_v8_0.h` | 31 | H | |
| `gmc_v9_0.c` | 2,381 | C | GFX9 (Vega) |
| `gmc_v9_0.h` | 31 | H | |
| `gmc_v10_0.c` | 1,152 | C | GFX10 (Navi/RDNA) |
| `gmc_v10_0.h` | 30 | H | |
| `gmc_v11_0.c` | 1,054 | C | GFX11 (RDNA2/RDNA3) |
| `gmc_v11_0.h` | 30 | H | |
| `gmc_v12_0.c` | 1,043 | C | GFX12 (Latest) |
| `gmc_v12_0.h` | 30 | H | |

### Buffer Objects (2,109 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_object.c` | 1,735 | C | Buffer object lifecycle |
| `amdgpu_object.h` | 374 | H | BO structures |

### TTM Integration (3,553 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_ttm.c` | 3,278 | C | TTM memory management |
| `amdgpu_ttm.h` | 275 | H | TTM structures |

### KFD Integration (3,674 lines)

| File | Lines | Type | Purpose |
|------|-------|------|---------|
| `amdgpu_amdkfd_gpuvm.c` | 3,674 | C | KFD compute context VM |

---

## Files Sorted by Size

1. `amdgpu_amdkfd_gpuvm.c` - 3,674 lines
2. `amdgpu_vm.c` - 3,285 lines
3. `amdgpu_ttm.c` - 3,278 lines
4. `gmc_v9_0.c` - 2,381 lines
5. `gmc_v8_0.c` - 1,783 lines
6. `amdgpu_object.c` - 1,735 lines
7. `amdgpu_gmc.c` - 1,692 lines
8. `gmc_v7_0.c` - 1,399 lines
9. `gmc_v6_0.c` - 1,171 lines
10. `gmc_v10_0.c` - 1,152 lines
11. `gmc_v11_0.c` - 1,054 lines
12. `gmc_v12_0.c` - 1,043 lines
13. `amdgpu_vm_pt.c` - 970 lines
14. `amdgpu_vm.h` - 710 lines
15. `amdgpu_gmc.h` - 471 lines
16. `amdgpu_object.h` - 374 lines
17. `amdgpu_ttm.h` - 275 lines
18. `amdgpu_vm_sdma.c` - 296 lines
19. `amdgpu_vm_cpu.c` - 122 lines
20. `amdgpu_vm_tlb_fence.c` - 111 lines
21-27. GMC version headers (29-31 lines each)

---

## Code Distribution by Category

```
GMC Version-Specific  ████████████████████████████████████████  41.1%
KFD Integration        ████████████████████████████              13.9%
TTM Integration        ███████████████████████████               13.2%
Core VM Management     ███████████████████████                   12.9%
GMC Core               ████████████                               7.9%
Buffer Objects         ████████                                   6.8%
Page Table Operations  ██                                         2.7%
Update Mechanisms      █                                          1.2%
TLB Management         ░                                          0.4%
```

---

## File Locations

All files are located in: `/workspace/drivers/gpu/drm/amd/amdgpu/`

### Evidence Commands

To verify line counts:
```bash
cd /workspace/drivers/gpu/drm/amd/amdgpu
wc -l amdgpu_vm*.c amdgpu_vm*.h amdgpu_gmc.c amdgpu_gmc.h \
     amdgpu_object.c amdgpu_object.h amdgpu_ttm.c amdgpu_ttm.h \
     amdgpu_amdkfd_gpuvm.c gmc_v*.c gmc_v*.h
```

To find all VM-related files:
```bash
find /workspace/drivers/gpu/drm/amd/amdgpu -name "*vm*" -o -name "*gmc*" | \
     grep -E "\.(c|h)$" | sort
```
