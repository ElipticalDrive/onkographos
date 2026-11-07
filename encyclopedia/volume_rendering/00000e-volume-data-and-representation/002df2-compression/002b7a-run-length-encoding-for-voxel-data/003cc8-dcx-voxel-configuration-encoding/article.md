# DCx Voxel Configuration Encoding

Also called: DCx Voxel Config Encoding

## Overview
DCx Voxel Configuration Encoding is a method for compactly representing the occupancy patterns of an "expanded cube" (a group of 8 voxels) within the Dual Contouring over Expanded Cubes (DCx) framework [S1]. It solves the problem of efficiently indexing and storing the 256 possible binary occupancy states by grouping rotationally and mirror-symmetric patterns into equivalence classes [S1]. It differs from its parent [Run-Length Encoding for Voxel Data](5051-run-length-encoding-for-voxel-data.md) by focusing on topological equivalence classes for mesh generation rather than run-length compression of raw data, and from siblings like [Chunk Serialization with RLE](5162-chunk-serialization-with-rle.md) by using a fixed 17-category lookup table based on geometric symmetry rather than variable-length run counts.

## How it works
**Inputs:** A binary occupancy mask for an expanded cube.
**Outputs:** An integer index (1–17) representing the equivalence class of the configuration.

**Data Structures:**
- **Expanded Cube Mask:** An 8-bit integer. Bit $i$ is 1 if voxel $i$ is active, 0 otherwise. The mapping of bit index $i$ to voxel position is implicit in the DCx grid structure.
- **Lookup Table (LUT):** A static table of 17 entries. Each entry corresponds to a representative category. Categories are labeled as $C-N$, where $C$ is the count of active voxels and $N$ is the serial index among categories with that count (e.g., 3-1, 4-2).

**Steps:**
1. **Input Validation:** Check if the number of active voxels (population count of the 8-bit mask) is less than 3. If so, the configuration is excluded from the LUT because it provides insufficient geometry for face construction [S1].
2. **Symmetry Grouping:** Identify the equivalence class of the input mask by applying all rotational and mirror symmetries of the cube to the mask. All masks in the same symmetry orbit map to the same category [S1].
3. **Category Selection:** Select the representative category from the 17 available classes. The categories are pre-computed and sorted by active voxel count, then by serial index [S1].
4. **Group Classification:** Determine if the category belongs to Group 1 or Group 2:
   - **Group 1:** Patterns with face generation rules that are self-contained and do not require information from neighboring expanded cubes [S1].
   - **Group 2:** Patterns that require adaptive face construction based on the categories of neighboring expanded cubes without generating the mesh immediately [S1].
5. **Output:** Return the category index (1–17).

**Parameters:**
- **Minimum Active Voxels:** 3. Configurations with fewer than 3 active voxels are excluded [S1].
- **LUT Size:** 17 representative categories [S1].

**Edge Cases and Pitfalls:**
- **Insufficient Geometry:** Configurations with <3 active voxels are invalid for this encoding scheme [S1].
- **Ambiguity in Group 2:** Group 2 patterns require context from neighbors, introducing ambiguity if neighbor categories are not resolved first [S1].
- **Non-Manifold Structures:** The LUT design aims to introduce non-manifold structures only when necessary to reduce holes, but this requires careful handling during mesh generation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_voxel_configuration(mask_8bit: int) -> int:
    """
    Encodes an 8-bit voxel occupancy mask into a DCx configuration category.
    
    Args:
        mask_8bit: 8-bit integer where bit i is 1 if voxel i is active.
        
    Returns:
        Integer category index (1-17).
        
    Raises:
        ValueError: If active voxel count < 3.
    """
    # Count active voxels
    active_count = bin(mask_8bit).count('1')
    
    # Exclude configurations with insufficient geometry
    if active_count < 3:
        raise ValueError("Configurations with fewer than 3 active voxels are excluded.")
    
    # Generate all symmetry-equivalent masks
    # This includes all 24 rotational symmetries and 24 mirror symmetries
    # of the cube applied to the 8-bit mask.
    # The specific bit-to-voxel mapping must be consistent with the DCx grid.
    equivalent_masks = generate_symmetry_equivalents(mask_8bit)
    
    # Find the canonical representative for this equivalence class
    # The canonical representative is the smallest integer in the equivalence class
    # that maps to one of the 17 pre-defined categories.
    canonical_mask = min(equivalent_masks)
    
    # Map canonical mask to category index
    # This requires a pre-computed lookup from canonical mask to category ID
    # The categories are labeled by (active_count, serial_index)
    category_index = get_category_from_canonical(canonical_mask, active_count)
    
    return category_index

def generate_symmetry_equivalents(mask: int) -> list:
    """
    Generates all 48 symmetry-equivalent masks for a given 8-bit mask.
    This is a placeholder for the specific symmetry operations defined in the DCx grid.
    """
    # Implementation depends on the specific voxel-to-bit mapping
    # and the symmetry group of the cube.
    # Not given in the sources.
    pass

def get_category_from_canonical(canonical_mask: int, active_count: int) -> int:
    """
    Maps a canonical mask to its category index (1-17).
    """
    # This requires a pre-computed table mapping canonical masks to category IDs.
    # The categories are sorted by active_count, then by serial index.
    # Not given in the sources.
    pass
```

## Subtopics

### [Feature Prediction Table Construction](9411-feature-prediction-table-construction.md)
Feature Prediction Table Construction is a preprocessing step in lossless voxel compression that builds a lookup table mapping local voxel contexts to predicted occupancy values. It reduces redundancy by determining the most frequent target voxel state for every possible configuration of neighboring voxels, enabling efficient residual encoding. It differs from Sparse Voxel Octree (SVO) approaches by modeling prediction contexts at full grid resolution with varying strides, rather than using local contexts of SVO nodes with different resolutions at each level.

## Operating characteristics
- **Time Complexity:** Encoding is $O(1)$ per expanded cube, assuming a pre-computed LUT and symmetry group operations are constant-time (48 symmetries).
- **Memory Complexity:** The LUT requires storage for 17 categories. The size of the encoded data is $\lceil \log_2(17) \rceil = 5$ bits per expanded cube, compared to 8 bits for the raw mask.
- **Strengths:** Significantly reduces redundancy by exploiting symmetry. Provides a compact representation for topological classification.
- **Limitations:** Excludes configurations with <3 active voxels. Group 2 patterns require neighbor context, complicating parallel processing.
- **When to use:** When using the DCx method for zero-level set extraction from neural unsigned distance functions and symmetry-based compression is beneficial.
- **When not to use:** When configurations with <3 active voxels are common or important.

## Background
DCx Voxel Configuration Encoding is part of the Dual Contouring over Expanded Cubes (DCx) method for zero-level set extraction from neural unsigned distance functions [S1]. It was introduced to reduce redundancy in the 256 possible occupancy patterns of an expanded cube by grouping symmetric patterns into 17 equivalence classes [S1]. The method aims to introduce non-manifold structures only when necessary to reduce holes in the resulting mesh [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
