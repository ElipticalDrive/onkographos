# Voxel Material Classification

Also called: ClassifyVoxel, Transparent Material Check

## Overview
Voxel Material Classification determines the optical and physical properties of a voxel to guide ray traversal and shading, specifically deciding whether a material is transparent (allowing rays to pass through) or opaque/solid [S1, unofficial]. It differs from [Voxel Filled Check](1705-voxel-filled-check.md), which only determines occupancy, by interpreting the material ID to return a semantic status (Air, Liquid, Solid) or transparency flag [S2, unofficial]. It is distinct from [Voxel Filtering](4035-voxel-filtering.md), which handles spatial smoothing or mipmapping, by focusing on discrete categorical classification rather than value interpolation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A packed voxel value (typically `uint32_t`) containing material identifiers [S2, unofficial].
*   Output: A classification status, either a boolean transparency flag [S1, unofficial] or an enumeration (`KnownAir`, `KnownLiquid`, `KnownSolid`, `UnknownBlocked`) [S2, unofficial].

**Data Structures:**
*   **Voxel:** A 32-bit integer where specific bits or ranges encode the material ID [S2, unofficial].
*   **Material Enum:** A discrete set of identifiers (e.g., `Utils::Material::Air`, `Utils::Material::Water`) [S2, unofficial].
*   **Status Enum:** `CollisionSampleStatus` with values `KnownAir`, `KnownSolid`, `KnownLiquid`, `UnknownBlocked` [S2, unofficial].

**Steps:**
1.  **Unpack Material:** Extract the material ID from the packed voxel. This is often delegated to a helper function like `Utils::UnpackMaterial` which returns a `uint8_t` material index [S2, unofficial].
2.  **Classify by Type (Method A - Semantic):** Compare the unpacked material against predefined semantic categories [S2, unofficial]:
    *   If material is `Air`, return `KnownAir`.
    *   If material is `Water`, `Lava`, or `Oil`, return `KnownLiquid`.
    *   Otherwise, return `KnownSolid`.
3.  **Classify by Transparency (Method B - Boolean):** Determine if the material allows ray traversal [S1, unofficial]. A material `m` is considered transparent if:
    *   $5 \le m \le 12$
    *   $m = 18$
    *   $m = 35$ (Specifically `MAT_TURF`, justified because blade cells are 90% air) [S1, unofficial].
    *   Otherwise, the material is opaque.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_voxel(voxel: uint32) -> str:
    """
    Classifies a voxel based on unpacked material ID.
    Returns 'AIR', 'LIQUID', or 'SOLID' based on [S2, unofficial].
    Also checks transparency logic from [S1, unofficial].
    """
    # Step 1: Unpack material
    # Implementation detail: Utils::UnpackMaterial extracts the relevant bits
    # from the uint32_t voxel.
    material_id = unpack_material(voxel)
    
    # Step 2: Semantic Classification (from [S2, unofficial])
    if material_id == MATERIAL_AIR:
        return "KnownAir"
    elif material_id in [MATERIAL_WATER, MATERIAL_LAVA, MATERIAL_OIL]:
        return "KnownLiquid"
    else:
        base_status = "KnownSolid"
        
    # Step 3: Transparency Check (from [S1, unofficial])
    # Returns True if transparent, False if opaque
    is_transparent = (
        (5 <= material_id <= 12) or 
        (material_id == 18) or 
        (material_id == 35)
    )
    
    # Note: The sources provide two distinct output styles.
    # Method A (S2) returns an enum. Method B (S1) returns a bool.
    # An implementation must choose one or map them.
    # If using S2's enum, transparency might be inferred or separate.
    # If using S1's bool, it overrides or supplements the semantic type.
    
    return base_status, is_transparent

def unpack_material(voxel: uint32) -> uint8:
    """
    Extracts material ID from packed voxel.
    Specific bit-packing scheme not detailed in sources [S2, unofficial].
    """
    # Placeholder for actual bit extraction logic defined in Utils::UnpackMaterial
    # [G] Assuming standard bit masking, e.g., voxel & 0xFF or similar.
    # The exact mask is not given in the sources.
    return voxel & 0xFF 
```

**Parameters:**
*   **Material 35 (`MAT_TURF`):** Explicitly treated as transparent because the geometry consists of blade cells that are 90% air [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bit Packing Ambiguity:** The exact bit layout of the `uint32_t` voxel is not specified in the sources [S2, unofficial]. Implementers must refer to the specific `Utils::UnpackMaterial` implementation in their codebase [S2, unofficial].
*   **Conflicting Outputs:** Source [S1, unofficial] defines a boolean transparency check, while Source [S2, unofficial] defines a semantic enum classification. These are not mutually exclusive but serve different pipeline stages (culling vs. shading). [G] Ensure the pipeline does not double-classify or conflict between the two if both are used.

## Subtopics

### [Voxel Material Palette Generation](3076-voxel-material-palette-generation.md)
Generates a compact lookup table (palette) mapping discrete voxel values (1–255) to PBR material definitions for six faces. It differs by batching materials into atlases to reduce memory overhead and draw calls, rather than storing per-voxel instances. Choose this when memory bandwidth is a bottleneck and materials are shared across many voxels.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel. The operation involves a constant-time unpack and a few integer comparisons [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space. No large tables are required for the classification logic itself, though the palette generation child method uses a 2D texture atlas [S1, unofficial].
*   **Strengths:** Extremely fast, suitable for real-time ray traversal culling [S1, unofficial].
*   **Limitations:** Relies on hardcoded material IDs and ranges, which may require updates if the voxel definition schema changes [S1, S2, unofficial].
*   **When to use:** During ray marching or traversal to skip empty space or handle liquid interactions [S1, S2, unofficial].
*   **When not to use:** When detailed physical properties (roughness, metalness) are needed immediately; use the palette generation method instead [S1, unofficial].

## Background
The method originates from voxel rendering engines that require efficient culling and shading of volumetric data. Source [S1, unofficial] references `voxelG`, an unofficial repository likely used for real-time voxel graphics [S1, unofficial]. Source [S2, unofficial] references `voxelrender`, another unofficial implementation focusing on chunk generation and simulation tests [S2, unofficial]. The distinction between "filled" (occupancy) and "material" (semantic/transparent) classification is a common optimization in voxel engines to separate coarse culling from fine-grained shading decisions.

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] asajid2-cell/voxelrender, https://github.com/asajid2-cell/voxelrender, fetched 2026-10-27 (unofficial)
