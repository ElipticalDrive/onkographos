# Voxel Material Classification

Also called: ClassifyVoxel, Transparent Material Check

## Overview
Voxel Material Classification determines whether a voxel material is transparent for ray traversal purposes, allowing rays to pass through [S1, unofficial]. It differs from [Voxel Filled Check](1705-voxel-filled-check.md), which only determines occupancy, by interpreting the unpacked material ID to return a semantic status or transparency flag [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A packed `uint32_t` voxel value [S2, unofficial].
*   Output: A classification status, either a boolean transparency flag [S1, unofficial] or an enumeration (`KnownAir`, `KnownLiquid`, `KnownSolid`, `UnknownBlocked`) [S2, unofficial].

**Data Structures:**
*   **Voxel:** A packed `uint32_t` integer containing material identifiers [S2, unofficial].
*   **Material Index:** An 8-bit unsigned integer (`uint8_t`) returned by the unpacking function [S2, unofficial].
*   **Status Enum:** `CollisionSampleStatus` with values `KnownAir`, `KnownSolid`, `KnownLiquid`, `UnknownBlocked` [S2, unofficial].

**Steps:**
1.  **Unpack Material:** Extract the material ID from the packed voxel using `Utils::UnpackMaterial`, which returns a `uint8_t` [S2, unofficial].
2.  **Classify by Type (Semantic):** Compare the unpacked material against predefined categories [S2, unofficial]:
    *   If material equals `Air`, return `KnownAir` [S2, unofficial].
    *   If material equals `Water`, `Lava`, or `Oil`, return `KnownLiquid` [S2, unofficial].
    *   Otherwise, return `KnownSolid` [S2, unofficial].
3.  **Classify by Transparency (Boolean):** Determine if the material allows ray traversal [S1, unofficial]. A material `m` is considered transparent if:
    *   $5 \le m \le 12$
    *   $m = 18$
    *   $m = 35$ (Specifically `MAT_TURF`, because blade cells are 90% air) [S1, unofficial].
    *   Otherwise, the material is opaque.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_voxel(voxel: int) -> tuple:
    # Step 1: Unpack material from packed uint32_t voxel [S2, unofficial]
    # [G] Exact bit-packing scheme not given in sources; assumes lowest 8 bits.
    material_id = voxel & 0xFF

    # Step 2: Semantic classification [S2, unofficial]
    # [G] Exact numeric values for material enums are implementation-specific.
    MATERIAL_AIR = 0
    MATERIAL_WATER = 1
    MATERIAL_LAVA = 2
    MATERIAL_OIL = 3

    if material_id == MATERIAL_AIR:
        status = "KnownAir"
    elif material_id in (MATERIAL_WATER, MATERIAL_LAVA, MATERIAL_OIL):
        status = "KnownLiquid"
    else:
        status = "KnownSolid"

    # Step 3: Transparency check [S1, unofficial]
    is_transparent = (5 <= material_id <= 12) or (material_id == 18) or (material_id == 35)

    return status, is_transparent
```

**Parameters:**
*   **Material 35 (`MAT_TURF`):** Explicitly treated as transparent because the geometry consists of blade cells that are 90% air [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bit Packing Ambiguity:** The exact bit layout of the `uint32_t` voxel is not specified in the sources [S2, unofficial]. Implementers must refer to the specific unpacking implementation in their codebase [S2, unofficial].
*   **Output Divergence:** Source [S1, unofficial] defines a boolean transparency check, while Source [S2, unofficial] defines a semantic enum classification. These serve different pipeline stages (culling vs. shading) and can be used together [G].

## Subtopics

### [Voxel Material Palette Generation](3076-voxel-material-palette-generation.md)
Generates a compact lookup table (palette) mapping discrete voxel values (1–255) to physically-based rendering (PBR) material definitions for each of the six voxel faces. It differs from per-voxel material instances by batching materials into atlases, thereby reducing memory overhead and draw calls. Choose this when memory bandwidth is a bottleneck and materials are shared across many voxels.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel. The operation involves a constant-time unpack and a few integer comparisons [G].
*   **Memory Complexity:** $O(1)$ auxiliary space. No large tables are required for the classification logic itself [G].
*   **Strengths:** Efficiently determines transparency for ray traversal culling [S1, unofficial].
*   **Limitations:** Relies on hardcoded material IDs and ranges [S1, S2, unofficial].
*   **When to use:** During ray marching or traversal to skip empty space or handle liquid interactions [S1, S2, unofficial].

## Background
The method originates from voxel rendering engines that require efficient culling and shading of volumetric data [G]. Source [S1, unofficial] documents the transparency logic for ray traversal [S1, unofficial]. Source [S2, unofficial] documents the semantic classification pipeline for chunk generation tests [S2, unofficial]. The distinction between "filled" (occupancy) and "material" (semantic/transparent) classification is a common optimization in voxel engines to separate coarse culling from fine-grained shading decisions [G].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] asajid2-cell/voxelrender, https://github.com/asajid2-cell/voxelrender, fetched 2026-10-27 (unofficial)
