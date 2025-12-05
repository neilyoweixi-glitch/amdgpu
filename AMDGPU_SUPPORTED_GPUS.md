# AMDGPU Supported GPUs

## Yes, AMDGPU Supports RDNA GPUs!

**AMDGPU fully supports AMD RDNA architecture GPUs**, including:

### RDNA 1 (Navi 1x) - GFX10
- **CHIP_NAVI10** (Radeon RX 5700 series)
  - PCI IDs: 0x7310, 0x7312, 0x7318, 0x7319, 0x731A, 0x731B, 0x731E, 0x731F
- **CHIP_NAVI12** (Radeon Pro W5700)
  - PCI IDs: 0x7360, 0x7362
- **CHIP_NAVI14** (Radeon RX 5500/5600 series)
  - PCI IDs: 0x7340, 0x7341, 0x7347, 0x734F

### RDNA 2 (Navi 2x) - GFX10.3
- **CHIP_SIENNA_CICHLID** (Radeon RX 6800/6900 series)
  - PCI IDs: 0x73A0, 0x73A1, 0x73A2, 0x73A3, 0x73A5, 0x73A8, 0x73A9, 0x73AB, 0x73AC, 0x73AD, 0x73AE, 0x73AF, 0x73BF
- **CHIP_NAVY_FLOUNDER** (Radeon RX 6700 series)
  - PCI IDs: 0x73C0, 0x73C1, 0x73C3, 0x73DA, 0x73DB, 0x73DC, 0x73DD, 0x73DE, 0x73DF
- **CHIP_DIMGREY_CAVEFISH** (Radeon RX 6600 series)
  - PCI IDs: 0x73E0, 0x73E1, 0x73E2, 0x73E3, 0x73E8, 0x73E9, 0x73EA, 0x73EB, 0x73EC, 0x73ED, 0x73EF, 0x73FF

### RDNA 3 (Navi 3x) - GFX11
- **CHIP_BEIGE_GOBY** (Radeon RX 7000 series)
  - PCI IDs: 0x7420, 0x7421, 0x7422, 0x7423, 0x7424, 0x743F

### RDNA APUs
- **CHIP_RENOIR** (Ryzen 4000 series APUs) - RDNA-based integrated graphics
  - PCI IDs: 0x15E7, 0x1636, 0x1638, 0x164C
- **CHIP_YELLOW_CARP** (Ryzen 5000 series APUs) - RDNA2-based integrated graphics
  - PCI IDs: 0x164D, 0x1681
- **CHIP_VANGOGH** (Steam Deck APU) - RDNA2-based
  - PCI ID: 0x163F

## Complete List of Supported GPU Families

### GCN 1.0 (Southern Islands - SI)
- CHIP_TAHITI (Radeon HD 7900 series)
- CHIP_PITCAIRN (Radeon HD 7800 series)
- CHIP_VERDE (Radeon HD 7700 series)
- CHIP_OLAND (Radeon HD 8600/8700 series)
- CHIP_HAINAN (Radeon HD 8600M series)

### GCN 2.0 (Sea Islands - CIK)
- CHIP_BONAIRE (Radeon HD 7790/R7 260 series)
- CHIP_KAVERI (APU - A10-7000 series)
- CHIP_KABINI (APU - A4/A6-7000 series)
- CHIP_HAWAII (Radeon R9 290/390 series)
- CHIP_MULLINS (APU - A4/A6/E2-6000 series)

### GCN 3.0 (Volcanic Islands - VI)
- CHIP_TOPAZ (Radeon R9 285)
- CHIP_TONGA (Radeon R9 380/380X)
- CHIP_FIJI (Radeon R9 Fury series)
- CHIP_CARRIZO (APU - A10-8000 series)
- CHIP_STONEY (APU - A9-9400 series)

### GCN 4.0 (Polaris)
- CHIP_POLARIS10 (Radeon RX 470/480/570/580)
- CHIP_POLARIS11 (Radeon RX 460/560)
- CHIP_POLARIS12 (Radeon RX 550)
- CHIP_VEGAM (Radeon RX Vega M)

### GCN 5.0 (Vega)
- CHIP_VEGA10 (Radeon RX Vega 56/64, Radeon Pro WX 9100)
- CHIP_VEGA12 (Radeon Pro WX 8200)
- CHIP_VEGA20 (Radeon VII, Radeon Pro VII)
- CHIP_RAVEN (Ryzen 2000/3000 series APUs)
- CHIP_ARCTURUS (Radeon Pro VII, Instinct MI50/MI60)

### RDNA 1 (Navi 1x) - GFX10
- CHIP_NAVI10 (Radeon RX 5700 series)
- CHIP_NAVI12 (Radeon Pro W5700)
- CHIP_NAVI14 (Radeon RX 5500/5600 series)

### RDNA 2 (Navi 2x) - GFX10.3
- CHIP_SIENNA_CICHLID (Radeon RX 6800/6900 series)
- CHIP_NAVY_FLOUNDER (Radeon RX 6700 series)
- CHIP_DIMGREY_CAVEFISH (Radeon RX 6600 series)
- CHIP_RENOIR (Ryzen 4000 series APUs)
- CHIP_YELLOW_CARP (Ryzen 5000 series APUs)
- CHIP_VANGOGH (Steam Deck APU)

### RDNA 3 (Navi 3x) - GFX11
- CHIP_BEIGE_GOBY (Radeon RX 7000 series)

### Data Center GPUs
- CHIP_ALDEBARAN (Instinct MI200 series)
- CHIP_CYAN_SKILLFISH (APU variants)

## Graphics Architecture Versions

The driver uses IP versioning to identify GPU capabilities:

- **GFX8**: GCN 4.0 (Polaris)
- **GFX9**: GCN 5.0 (Vega)
- **GFX10**: RDNA 1 (Navi 1x)
- **GFX10.3**: RDNA 2 (Navi 2x)
- **GFX11**: RDNA 3 (Navi 3x)
- **GFX12**: Latest generation (likely RDNA 4)

## Page Table Support

All RDNA GPUs use the same GPUVM page table management system:
- Multi-level page tables (up to 5 levels)
- Support for both CPU and SDMA-based updates
- Variable page sizes (fragments)
- ASIC-specific PTE encoding (GFX10 vs GFX11 vs GFX12)

## Notes

1. **IP Discovery**: Newer GPUs may use IP discovery instead of hardcoded PCI IDs
2. **APU Support**: Many APUs with integrated RDNA graphics are supported
3. **Experimental Hardware**: Some newer GPUs may require `amdgpu.exp_hw_support=1` module parameter
4. **Kernel Version**: Full support for latest GPUs requires recent kernel versions

## References

- PCI ID table: `drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c`
- Chip definitions: `include/drm/amd_asic_type.h`
- Page table code: `drivers/gpu/drm/amd/amdgpu/amdgpu_vm*.c`
