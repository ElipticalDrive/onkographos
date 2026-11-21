# DCx Voxel Configuration Encoding

Also called: DCx LUT, Expanded Cube Configuration Encoding

## Overview
DCx Voxel Configuration Encoding is a method for compactly representing the occupancy state of an expanded cube (8 voxels) within the Dual Contouring over Expanded Cubes (DCx) framework. It maps the 256 possible binary occupancy patterns to a small set of 17 representative equivalence classes, enabling efficient face construction and topology management. It differs from general run-length encoding by exploiting geometric symmetry and topological constraints specific to isosurface extraction, rather than compressing sequential data values.

## How it works
**Inputs and Outputs:**
- Input: An 8-bit binary mask representing the occupancy of 8 voxels in an expanded cube.
- Output: An integer index (0–16) identifying the equivalence class of the pattern.

**Data Structures:**
- **Binary Mask:** An 8-bit integer where bit $i$ is 1 if voxel $i$ is active, 0 otherwise.
- **Lookup Table (LUT):** A mapping from the 17 equivalence classes to their properties. The LUT size is fixed at 17 entries. Each entry is labeled by a category string (e.g., "3-1") where the first number is the count of active voxels and the second is the serial index among cases with that count.

**Steps:**
1. **Mask Generation:** Construct an 8-bit integer $M$ from the occupancy status of the 8 voxels in the expanded cube.
2. **Symmetry Grouping:** Group the 256 possible patterns into equivalence classes based on rotational and mirror symmetry. Patterns that can be transformed into each other via symmetry operations are considered equivalent.
3. **Filtering:** Exclude all configurations with fewer than 3 active voxels. These are excluded because they provide insufficient geometry for face construction [S1].
4. **Classification:** Map the remaining patterns to the 17 representative categories. The categories are labeled by voxel count and serial number (e.g., 3-1, 4-2).
5. **Group Assignment:** Assign each category to one of two groups:
   - **Group 1:** Patterns with self-contained face generation rules that do not require information from neighboring expanded cubes.
   - **Group 2:** Patterns requiring adaptive face construction based on the categories of neighboring expanded cubes, without generating the mesh immediately.

**Formulas:**
- Total patterns: $2^8 = 256$.
- Active voxel count: $N_{active} = \sum_{i=0}^{7} M_i$.
- Exclusion condition: If $N_{active} < 3$, the pattern is excluded.

**Parameters:**
- Minimum active voxel count: 3 [S1].
- LUT Size: 17 representative categories [S1].

**Edge Cases and Pitfalls:**
- Patterns with fewer than 3 active voxels are invalid for face construction and must be handled or excluded [S1].
- Group 2 patterns introduce ambiguity requiring context from neighbors, which must be resolved during mesh generation.

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
    # Count active voxels
    active_count = bin(voxel_mask).count('1')
    
    # Exclude configurations with insufficient geometry
    if active_count < 3:
        raise ValueError("Configuration has fewer than 3 active voxels.")
    
    # Apply symmetry reduction and map to LUT index
    # The exact mapping logic depends on the specific symmetry groupings defined in the source.
    # This is a placeholder for the actual LUT lookup logic.
    # In practice, this would involve checking the mask against precomputed symmetry classes.
    
    # For the purpose of this implementation, we assume a direct mapping exists.
    # The actual implementation would require the full symmetry group logic.
    # Since the source does not provide the exact mapping algorithm, we return a placeholder.
    # [G] The source mentions 17 categories but does not detail the exact mapping function.
    # A complete implementation would require the full symmetry group definitions.
    
    # Placeholder return value. In a real implementation, this would be the LUT index.
    return 0  # Placeholder
```

## Subtopics

### [Feature Prediction Table Construction](9411-feature-prediction-table-construction.md)
Builds a compact lookup table mapping neighboring voxel occupancy to predicted occupancy, reducing redundancy before sparse encoding. It uses multi-level context-aware prediction based on uniformly spaced key voxels, differing from local SVO node contexts.

## Operating characteristics
- **Time Complexity:** Encoding is $O(1)$ per expanded cube, as it involves a fixed number of bit operations and a LUT lookup.
- **Memory Complexity:** The LUT size is fixed at 17 entries, making memory usage negligible.
- **Strengths:** Significantly reduces redundancy by grouping symmetric patterns. Excludes topologically invalid configurations, simplifying face construction.
- **Limitations:** Requires handling of Group 2 patterns, which introduce ambiguity and require neighbor context. The exact symmetry grouping logic is not fully detailed in the source.
- **When to Use:** Suitable for DCx-based isosurface extraction where compact representation of voxel configurations is needed.
- **When Not to Use:** Not suitable for general voxel data compression where symmetry is not a relevant factor.

## Background
DCx Voxel Configuration Encoding was introduced in the context of Dual Contouring over Expanded Cubes (DCx) for zero-level set extraction from neural unsigned distance functions. It was developed to address the need for efficient and robust face construction in isosurface extraction, leveraging geometric symmetry to reduce the complexity of handling 256 possible occupancy patterns.

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
