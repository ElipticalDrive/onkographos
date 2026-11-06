# Choosing Alternative Resolutions

Also called: Choosing an alternate resolution given a node’s bitmap, Finding Lower Resolution Levels

## Overview
This method selects a cache-resident resolution level for a volume rendering brick when the desired resolution is not available in memory [S1, unofficial]. It leverages a bitmap of partially mapped resolutions stored in each residency-octree node to efficiently identify candidate levels without random searching [S1, unofficial]. The approach prioritizes lower resolutions before attempting higher ones, iterating through candidates until a mapped page is found or all options are exhausted [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `node.bitmap`: A bitmask where each bit $i$ indicates if resolution level $i$ is partially mapped in the node. `resolutionLevel`: The index of the desired resolution level (which is currently unmapped).
*   **Output:** The index of a resolution level that is both partially mapped in the node and cache-resident for the current ray, or `NONE` if no such level exists.

**Data Structures:**
*   **Resolution Ordering:** Resolution levels are indexed from $0$ (highest resolution) to $k$ (lowest resolution) [S1, unofficial].
*   **Bitmap:** An integer or bit-vector where the $i$-th bit corresponds to resolution level $i$ [S1, unofficial].

**Algorithm Steps:**
1.  **Attempt Lower Resolutions:**
    *   Compute the next candidate resolution by calling `nextLower(node.bitmap, resolutionLevel)`.
    *   Retrieve the page for this resolution using `getPage(node, ray, resolutionLevel)`.
    *   While the candidate resolution is valid (`!= NONE`) and the page is unmapped (`page.unmapped` is true):
        *   Update the candidate to the next lower resolution using `nextLower(node.bitmap, resolutionLevel)`.
        *   Retrieve the new page.
    *   If the loop terminates because `page.unmapped` is false, the current `nextResolution` is the result.
2.  **Attempt Higher Resolutions (if lower failed):**
    *   If the previous step resulted in an unmapped page (either because no lower resolutions were available or all were unmapped):
        *   Compute the next candidate resolution by calling `nextHigher(node.bitmap, resolutionLevel)`.
        *   Retrieve the page for this resolution.
        *   While the candidate resolution is valid (`!= NONE`) and the page is unmapped:
            *   Update the candidate to the next higher resolution using `nextHigher(node.bitmap, resolutionLevel)`.
            *   Retrieve the new page.
    *   If the loop terminates because `page.unmapped` is false, the current `nextResolution` is the result.
    *   If no valid resolution is found, return `NONE`.

**Helper Functions:**

*   `nextLower(partiallyMapped, resolution)`:
    1.  Compute a mask of all resolution levels strictly lower than `resolution`. This is done by calling `getLowerResolutionsMask(resolution)`, which returns a constant mask with bits set for all indices $j < resolution$.
    2.  Compute `lower = partiallyMapped AND getLowerResolutionsMask(resolution)`. This isolates bits for resolutions that are both partially mapped and lower than the current level.
    3.  If `lower == 0`, return `NONE`.
    4.  Return the index of the first trailing one-bit of `lower`. Since resolution $0$ is highest and $k$ is lowest, the "first trailing one-bit" (least significant bit set) corresponds to the highest index among the lower resolutions, which is the *next* lower resolution in terms of detail (coarser) [S1, unofficial].
    *   *Implementation Note:* If `firstTrailingBit` is unavailable, compute the index as the number of consecutive trailing zero-bits from the least significant bit (LSB) [S1, unofficial].

*   `nextHigher(partiallyMapped, resolution)`:
    *   Similar to `nextLower`, but uses `getHigherResolutionsMask(resolution)` to mask bits for indices $j > resolution$. The result is the index of the first trailing one-bit of the masked value, which corresponds to the next higher resolution (finer) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def choose_alternative_resolution(node_bitmap, resolution_level):
    """
    Selects a cache-resident resolution level for a brick.
    
    Args:
        node_bitmap (int): Bitmask of partially mapped resolutions.
        resolution_level (int): Index of the desired (unmapped) resolution.
        
    Returns:
        int: Index of a valid, mapped resolution level, or None if none found.
    """
    
    # Step 1: Try lower resolutions (coarser)
    next_res = next_lower(node_bitmap, resolution_level)
    page = get_page(node_bitmap, resolution_level) # Placeholder for actual page retrieval logic
    
    while next_res is not None and page.unmapped:
        next_res = next_lower(node_bitmap, resolution_level)
        page = get_page(node_bitmap, next_res) # Update page for new resolution
        
    if not page.unmapped:
        return next_res
        
    # Step 2: Try higher resolutions (finer) if lower failed
    next_res = next_higher(node_bitmap, resolution_level)
    page = get_page(node_bitmap, resolution_level) # Placeholder
    
    while next_res is not None and page.unmapped:
        next_res = next_higher(node_bitmap, resolution_level)
        page = get_page(node_bitmap, next_res) # Update page for new resolution
        
    if not page.unmapped:
        return next_res
        
    return None

def next_lower(partially_mapped, resolution):
    """
    Finds the next lower resolution level (higher index) that is partially mapped.
    """
    # Get mask for all resolutions strictly lower than 'resolution'
    # Since 0 is highest, k is lowest, "lower" means index > resolution
    lower_mask = get_lower_resolutions_mask(resolution)
    
    # Intersect with partially mapped bits
    lower = partially_mapped & lower_mask
    
    if lower == 0:
        return None
        
    # Return index of first trailing one-bit (LSB)
    # This corresponds to the smallest index > resolution that is set
    return first_trailing_bit(lower)

def next_higher(partially_mapped, resolution):
    """
    Finds the next higher resolution level (lower index) that is partially mapped.
    """
    # Get mask for all resolutions strictly higher than 'resolution'
    # "Higher" means index < resolution
    higher_mask = get_higher_resolutions_mask(resolution)
    
    # Intersect with partially mapped bits
    higher = partially_mapped & higher_mask
    
    if higher == 0:
        return None
        
    # Return index of first trailing one-bit (LSB)
    # This corresponds to the largest index < resolution that is set
    return first_trailing_bit(higher)

def first_trailing_bit(value):
    """
    Returns the index of the least significant bit that is set to 1.
    Equivalent to the number of trailing zeros.
    """
    if value == 0:
        return None
    # Implementation depends on architecture, e.g., ctz (count trailing zeros)
    # Here we assume a standard bit manipulation function
    return (value & -value).bit_length() - 1 # Python-specific trick for LSB index

def get_lower_resolutions_mask(resolution):
    """
    Returns a mask with bits set for all indices j > resolution.
    Assumes a fixed maximum resolution k.
    """
    # Not given in the sources. Typically this is a precomputed constant or
    # computed as ((1 << (k+1)) - 1) ^ ((1 << (resolution+1)) - 1)
    # where k is the maximum resolution index.
    raise NotImplementedError("Mask generation depends on system-specific max resolution k")

def get_higher_resolutions_mask(resolution):
    """
    Returns a mask with bits set for all indices j < resolution.
    """
    # Not given in the sources. Typically this is (1 << resolution) - 1
    return (1 << resolution) - 1

def get_page(node_bitmap, resolution):
    """
    Retrieves the page object for the given resolution.
    """
    # Not given in the sources. This function interacts with the memory manager.
    raise NotImplementedError("Page retrieval depends on memory manager implementation")
```

**Parameters:**
*   **Resolution Ordering:** $0$ is the highest resolution, $k$ is the lowest [S1, unofficial].
*   **Mask Generation:** The masks for lower and higher resolutions are derived from constant arrays or bit shifts based on the maximum resolution $k$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **No Mapped Resolutions:** If the bitmap is zero or no bits correspond to valid resolutions relative to the current level, the function returns `NONE` [S1, unofficial].
*   **Unmapped Regions:** The bitmap only indicates if a node is *partially* mapped. A ray sample may still fall into an unmapped region within a partially mapped brick. The algorithm iterates through candidates to handle this [S1, unofficial].
*   **Bit Manipulation:** If `firstTrailingBit` is not available, the index must be computed as the number of consecutive trailing zero-bits from the LSB [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The bitmap operations (`AND`, `firstTrailingBit`) are $O(1)$ or $O(\log k)$ depending on implementation. The iteration depends on the number of unmapped resolutions, which is typically small.
*   **Memory Complexity:** $O(1)$ auxiliary space per node, as it relies on the existing bitmap.
*   **Strengths:** Efficiently finds a suitable resolution without searching all levels. Uses existing residency information.
*   **Limitations:** May still fail if no resolution is mapped for the specific ray sample. The strategy of trying lower then higher is arbitrary; other strategies (higher then lower, alternating, closest) exist [S1, unofficial].

## Background
This method is part of mixed-resolution multi-volume rendering, specifically within the context of residency-octrees and cache-aware rendering [S1, unofficial]. It was described in the work by Faludi et al. [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
