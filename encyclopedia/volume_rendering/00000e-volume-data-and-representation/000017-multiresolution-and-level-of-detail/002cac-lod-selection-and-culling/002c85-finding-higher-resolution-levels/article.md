# Finding Higher Resolution Levels

Also called: Choosing a higher resolution given a node’s bitmap

## Overview
This method finds the next higher resolution level in a residency octree traversal where a node is partially mapped [S1, unofficial]. It returns the index of the finest available resolution level strictly higher than a given resolution index, based on a node's occupancy bitmap [S1, unofficial]. It differs from sibling methods that generate data or make final rendering decisions by performing a specific bitwise lookup during hierarchical traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   `partiallyMapped`: A bitmask where the $j$-th bit is set to 1 if the node is partially mapped at resolution level $j$, and 0 otherwise [S1, unofficial].
*   `resolution`: The integer index of the resolution level last tested [S1, unofficial].
*   **Output**: The integer index of the next higher resolution level, or a special value `NONE` if no higher partially mapped resolution exists [S1, unofficial].

**Data Structures:**
*   **Resolution Indexing**: Resolution levels are ordered from 0 (highest) to $k$ (lowest) [S1, unofficial]. Consequently, "higher resolution" corresponds to a lower numerical index [S1, unofficial].
*   **Bitmask**: The `partiallyMapped` variable is an integer of fixed bit-width `BIT_WIDTH`, sufficient to cover all possible resolution levels in the hierarchy [S1, unofficial].

**Steps:**
1.  **Mask Generation**: Compute a mask `higher` by performing a bitwise AND between `partiallyMapped` and a precomputed mask that isolates bits corresponding to resolution levels strictly higher than the given `resolution` index [S1, unofficial].
2.  **Empty Check**: If the resulting `higher` mask is 0, it means the node is not partially mapped in any resolution level higher than the current one [S1, unofficial]. Return `NONE` [S1, unofficial].
3.  **Index Extraction**: If `higher` is non-zero, the index of the next higher resolution is the position of the first leading one-bit in `higher` [S1, unofficial].

**Formulas:**
If the hardware instruction `firstLeadingBit` is unavailable, the index $i$ of the first leading one-bit of an integer $x$ can be computed using the `countLeadingZeros` function:
$$ i = \text{BIT\_WIDTH} - 1 - \text{countLeadingZeros}(x) $$
where:
*   $i$ is the index of the first leading one-bit [S1, unofficial].
*   $\text{BIT\_WIDTH}$ is the total number of bits in the integer type of $x$ [S1, unofficial].
*   $\text{countLeadingZeros}(x)$ returns the number of zero bits preceding the most significant one-bit in $x$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_higher_resolution(partially_mapped: int, resolution: int, bit_width: int) -> int:
    """
    Finds the index of the next higher resolution level where the node is partially mapped.
    
    Args:
        partially_mapped: Bitmask of resolutions where the node is partially mapped.
        resolution: Index of the resolution level last tested.
        bit_width: Total bit-width of the integer type used for the mask.
        
    Returns:
        Index of the next higher resolution, or -1 (NONE) if none exists.
    """
    # Step 1: Create a mask for all resolutions strictly higher than 'resolution'.
    # Since index 0 is highest, higher resolutions have indices < resolution.
    # (1 << resolution) - 1 creates a mask with 1s at positions 0 to resolution-1.
    higher_mask = (1 << resolution) - 1
    higher = partially_mapped & higher_mask
    
    # Step 2: Check if any higher resolution is partially mapped
    if higher == 0:
        return -1  # NONE
        
    # Step 3: Find the index of the first leading one-bit.
    # Using the portable fallback formula from the sources [S1, unofficial].
    # countLeadingZeros(x) for a fixed width W is W - x.bit_length() for x > 0.
    leading_zeros = bit_width - higher.bit_length()
    i = bit_width - 1 - leading_zeros
    
    return i
```

**Parameters:**
*   `bit_width`: The bit-width of the integer type used for the mask [S1, unofficial]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero Mask**: If `higher` is 0, the algorithm must return `NONE` [S1, unofficial].
*   **Resolution Ordering**: The algorithm relies on the convention that resolution index 0 is the highest resolution [S1, unofficial].
*   **Hardware Support**: The `firstLeadingBit` instruction is not available on all platforms, requiring the `countLeadingZeros` fallback [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity**: Not given in the sources.
*   **Strengths**: Extremely fast bitwise operation suitable for real-time traversal [G].
*   **Limitations**: Requires the `partiallyMapped` bitmap to be precomputed and maintained [S1, unofficial].
*   **When to use**: During residency octree traversal when determining the next level of detail to sample [S1, unofficial].
*   **When not to use**: When the hierarchy is not based on a fixed set of resolution levels indexed by integers, or when the "partially mapped" status is not tracked via a bitmap [G].

## Background
This method is described in the context of scalable web-based volume rendering of large biomedical datasets [S1, unofficial]. It is part of a residency octree traversal algorithm designed to dynamically select resolution levels based on partial occupancy [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
