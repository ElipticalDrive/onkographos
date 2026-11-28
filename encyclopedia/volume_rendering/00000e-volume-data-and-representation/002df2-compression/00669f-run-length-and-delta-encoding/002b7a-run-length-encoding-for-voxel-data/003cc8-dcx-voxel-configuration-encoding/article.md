# DCx Voxel Configuration Encoding


## Overview
DCx Voxel Configuration Encoding compactly represents the occupancy state of an expanded cube (8 voxels) within the Dual Contouring over Expanded Cubes (DCx) framework [S1]. It maps the 256 possible binary occupancy patterns to 17 representative equivalence classes, enabling efficient face construction and topology management [S1]. Unlike general run-length encoding, it exploits geometric symmetry and topological constraints specific to isosurface extraction rather than compressing sequential data values [G].

## How it works
**Inputs and Outputs:**
- Input: An 8-bit binary mask representing the occupancy of 8 voxels in an expanded cube [S1].
- Output: An integer index (0–16) identifying the equivalence class of the pattern [S1].

**Data Structures:**
- **Binary Mask:** An 8-bit integer where bit $i$ is 1 if voxel $i$ is active, 0 otherwise [S1].
- **Lookup Table (LUT):** A mapping from the 17 equivalence classes to their properties [S1]. The LUT size is fixed at 17 entries [S1]. Each entry is labeled by a category string (e.g., "3-1") where the first number is the count of active voxels and the second is the serial index among cases with that count [S1].

**Steps:**
1. **Mask Generation:** Construct an 8-bit integer $M$ from the occupancy status of the 8 voxels in the expanded cube [S1].
2. **Symmetry Grouping:** Group the 256 possible patterns into equivalence classes based on rotational and mirror symmetry [S1]. Patterns that can be transformed into each other via symmetry operations are considered equivalent [S1].
3. **Filtering:** Exclude all configurations with fewer than 3 active voxels because they provide insufficient geometry for face construction [S1].
4. **Classification:** Map the remaining patterns to the 17 representative categories [S1]. The categories are labeled by voxel count and serial number (e.g., 3-1, 4-2) [S1].
5. **Group Assignment:** Assign each category to one of two groups [S1]:
   - **Group 1:** Patterns with self-contained face generation rules that do not require information from neighboring expanded cubes [S1].
   - **Group 2:** Patterns requiring adaptive face construction based on the categories of neighboring expanded cubes, without generating the mesh immediately [S1].

**Formulas:**
- Total patterns: $2^8 = 256$ [S1].
- Active voxel count: $N_{\text{active}} = \sum_{i=0}^{7} M_i$ [G].
- Exclusion condition: If $N_{\text{active}} < 3$, the pattern is excluded [S1].

**Parameters:**
- Minimum active voxel count: 3 [S1].
- LUT Size: 17 representative categories [S1].

**Edge Cases and Pitfalls:**
- Patterns with fewer than 3 active voxels are invalid for face construction and must be handled or excluded [S1].
- Group 2 patterns introduce ambiguity requiring context from neighbors, which must be resolved during mesh generation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_voxel_configuration(voxel_mask: int) -> int:
    """
    Encodes an 8-bit voxel mask into a DCx configuration index.
    
    Args:
        voxel_mask: An 8-bit integer where bit i is 1 if voxel i is active.
        
    Returns:
        An integer index (0-16) representing the equivalence class.
        
    Raises:
        ValueError: If the mask has fewer than 3 active voxels.
    """
    # Mask Generation: Construct an 8-bit integer M from the occupancy status.
    M = voxel_mask & 0xFF
    
    # Filtering: Exclude configurations with fewer than 3 active voxels.
    active_count = bin(M).count('1')
    if active_count < 3:
        raise ValueError("Configuration has fewer than 3 active voxels.")
        
    # Symmetry Grouping & Classification: Map to 17 representative categories.
    # [G] The exact symmetry equivalence mapping algorithm is not provided in the sources.
    # A complete implementation requires a precomputed lookup table mapping each valid 8-bit mask to one of 17 indices.
    # In practice, this step performs: return LUT[M]
    return 0  # Returns the LUT index (0-16) for the given mask
```

## Subtopics

### [Feature Prediction Table Construction](9411-feature-prediction-table-construction.md)
Builds a compact lookup table mapping neighboring voxel occupancy to predicted occupancy, reducing redundancy before sparse encoding. It uses multi-level context-aware prediction based on uniformly spaced key voxels, differing from local SVO node contexts.

## Operating characteristics
- **Time Complexity:** Encoding is $O(1)$ per expanded cube, as it involves a fixed number of bit operations and a LUT lookup [G].
- **Memory Complexity:** The LUT size is fixed at 17 entries, making memory usage negligible [S1].
- **Strengths:** Significantly reduces redundancy by grouping symmetric patterns [S1]. Excludes topologically invalid configurations, simplifying face construction [S1].
- **Limitations:** Requires handling of Group 2 patterns, which introduce ambiguity and require neighbor context [S1]. The exact symmetry grouping logic is not fully detailed in the source [G].
- **When to Use:** Suitable for DCx-based isosurface extraction where compact representation of voxel configurations is needed [G].
- **When Not to Use:** Not suitable for general voxel data compression where symmetry is not a relevant factor [G].

## Background
DCx Voxel Configuration Encoding was introduced in the context of Dual Contouring over Expanded Cubes (DCx) for zero-level set extraction from neural unsigned distance functions [S1]. It was developed to address the need for efficient and robust face construction in isosurface extraction, leveraging geometric symmetry to reduce the complexity of handling 256 possible occupancy patterns [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
