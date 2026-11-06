# Voxel Data Structure

Also called: VoxelData

## Overview
The Voxel Data Structure stores per-voxel attributes in a compact memory layout to address bandwidth and capacity constraints in dense volume rendering and voxel engines [S2, unofficial]. It differs from sibling structures like [Voxel Data Layout](4076-voxel-data-layout.md) by offering variants that trade attribute richness for size, such as bit-packed formats or floating-point density fields [S1, S2, S3, unofficial].

## How it works
The core method involves defining a struct that holds voxel attributes [S1, S2, unofficial]. Two primary implementations exist in the source literature, differing in attribute selection and size [S1, S2, unofficial].

### Inputs and Outputs
*   **Input:** Voxel coordinates (implicit or explicit), material properties, density values, and geometric normals [S1, S2, unofficial].
*   **Output:** A contiguous memory block of voxel structs, accessible via linear indexing or spatial hashing [G].

### Data Structures and Layouts
1.  **Rich Attribute Layout [S1, unofficial]:**
    *   `morton`: `uint64_t` (64 bits). Stores the Morton code for spatial ordering [S1, unofficial].
    *   `color`: `vec3` (3 floats, 96 bits). RGB color data [S1, unofficial].
    *   `normal`: `vec3` (3 floats, 96 bits). Surface normal data [S1, unofficial].
    *   **Total Size:** $8 + 3 \times 4 + 3 \times 4 = 32$ bytes per voxel [S1, unofficial].
    *   **Comparison:** Operators `>` and `<` are defined based on the `morton` code value [S1, unofficial].

2.  **Compact Density Layout [S2, unofficial]:**
    *   `density`: `half` (fp16, 16 bits). Density representation [S2, unofficial].
    *   `materialId`: `uint` or `int` (32 bits). Identifier for the material type [S2, unofficial].
    *   `flags`: `bool` or `uint` (variable, typically packed to fit 4 bytes total) [S2, unofficial].
    *   **Total Size:** 4 bytes per voxel [S2, unofficial].
    *   **Precision:** Uses half-precision (fp16) for density to balance precision and memory usage [S2, unofficial].

### Numbered Steps for Implementation
1.  **Define the Struct:** Choose the layout based on the rendering or meshing requirements [S2, unofficial].
2.  **Initialize Memory:** Allocate a flat array of the chosen struct type [G].
3.  **Set Voxel Data:** Use factory methods to populate the array: `CreateSolid(materialId)` for blocky voxels or `CreateSmooth(density, materialId)` for terrain [S2, unofficial].
4.  **Access:** Retrieve voxel data using a linear index $i = x + y \cdot W + z \cdot W \cdot H$ [G].

### Formulas
*   **Chunk Memory ($32^3$):** Half precision density results in 68 KB per $32^3$ chunk, compared to 128 KB for single precision [S2, unofficial].
    $$ \text{Size}_{\text{fp16}} \approx 0.5 \times \text{Size}_{\text{fp32}} $$
*   **Memory Savings:** Using fp16 for density reduces memory usage by 50% compared to fp32 [S2, unofficial].

### Parameters
*   **Density Precision:** `half` (fp16) [S2, unofficial].
*   **Chunk Size:** Default $32^3$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision Loss:** fp16 has limited dynamic range and precision, which may cause artifacts in high-contrast density fields [S2, unofficial].
*   **Alignment:** The 4-byte layout requires careful packing of `materialId` and `flags` to avoid compiler-inserted padding [S2, unofficial].
*   **Morton Code Overflow:** In the 32-byte layout, `morton` codes must fit in `uint64_t` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def float_to_half(f):
    # Standard IEEE 754 half-precision conversion [G]
    return struct.pack('<e', f)

class VoxelDataCompact:
    density: bytes  # fp16
    material_id: int
    flags: int

    @staticmethod
    def create_solid(material_id: int) -> 'VoxelDataCompact':
        return VoxelDataCompact(density=float_to_half(0.0), material_id=material_id, flags=0)

    @staticmethod
    def create_smooth(density: float, material_id: int) -> 'VoxelDataCompact':
        return VoxelDataCompact(density=float_to_half(density), material_id=material_id, flags=0)

class VoxelDataRich:
    morton: int
    color: tuple
    normal: tuple

    def __gt__(self, other: 'VoxelDataRich') -> bool:
        return self.morton > other.morton

    def __lt__(self, other: 'VoxelDataRich') -> bool:
        return self.morton < other.morton
```

## Subtopics

### 64-bit Bit-packed Voxel Format
A 64-bit packed structure storing voxel metadata using bit fields, accessed via template-based bit set/get utilities [S3, unofficial]. It stores Voxel Index (bits 0–15, 16 bits), State Bit (bit 15, 1 bit), Type Info (bits 16–31, 16 bits), and User Data (bits 32–63, 32 bits) in a single `uint64_t` [S3, unofficial]. Bit manipulation uses `util::bset<beg_val, size_val>(n_, value_)` which applies `n_ = (n_ & cmask_) | vmask_;` where `cmask_ = ~(width_ << beg_val)` and `vmask_ = (value_ & width_) << beg_val` [S3, unofficial]. A pitfall is that the State Bit (bit 15) overlaps with the Voxel Index field (bits 0–15), which may cause data corruption if both are written independently without masking [S3, unofficial].

## Operating characteristics
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels [G]. The 4-byte layout uses 4 bytes/voxel, while the 32-byte layout uses 32 bytes/voxel [S1, S2, unofficial].
*   **Performance:** The compact layout enables higher cache efficiency and reduced memory bandwidth usage, crucial for real-time meshing jobs [S2, unofficial].
*   **Strengths:** Minimal memory footprint (4 bytes/voxel) allows for larger world chunks or higher resolution grids within fixed memory budgets [S2, unofficial].
*   **Limitations:** The 4-byte layout lacks per-voxel normals and colors, requiring external data structures or interpolation for shading [S2, unofficial]. The 32-byte layout stores more attributes but increases memory consumption [S1, unofficial].
*   **When to Use:** Use the 4-byte layout for density-based terrain and blocky meshing [S2, unofficial]. Use the 32-byte layout when per-voxel shading data is required [S1, unofficial].

## Background
The concept of compact voxel data structures emerged with the need for efficient voxel-based terrain and volume rendering in real-time applications. The 4-byte layout was implemented in the "StraightFour" voxel engine to support generic, terrain, and smoothed voxel features using Unity's Job System and Burst compiler [S2, unofficial]. The 32-byte layout with Morton codes is associated with CPU-based voxel raycasting implementations [S1, unofficial]. The 64-bit bit-packed format is used in spatial voxel octree implementations [S3, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S3] SeveriSuominen/rapid-svo, https://github.com/SeveriSuominen/rapid-svo, fetched 2026-10-26 (unofficial)
