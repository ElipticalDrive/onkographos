# Data Fusion via Channel Merging

Also called: VConv Merging

## Overview
Data Fusion via Channel Merging is a preprocessing technique that combines multiple data channels (variables or modalities) into a single unified volume file for multi-variable volume rendering [S1, unofficial]. It solves the problem of managing separate data files for different modalities, which makes it difficult to distinguish intensity overlaps without significant user expertise in color component composition [S1, unofficial]. It sets itself apart from siblings like [Byte/Short Conversion](11742-byte-short-conversion.md) (which handles type conversion) and [Intensity/Value Range Transformation](17795-intensity-value-range-transformation.md) (which handles scaling) by physically concatenating distinct variable arrays into a single voxel structure [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Multiple source volume datasets (channels), each potentially containing different variables (e.g., scalar values, gradients, Hessians) or modalities.
*   **Output:** A single volume file where each voxel contains a concatenated array of elements from all input channels.

**Data Structures:**
The output volume stores data in a row-major order with the z-axis as the outer loop, y as the middle, and x as the inner loop [S2, unofficial]. Each voxel stores `ne` elements, where `ne` is the total number of elements per voxel across all merged channels [S2, unofficial].

**Steps:**
1.  Determine the number of elements per voxel (`nelts`) for the primary volume.
2.  Calculate the total elements per voxel (`ne`) by summing `nelts` from the primary volume and all additional input volumes. If gradient (`addG`) or Hessian (`addH`) data is to be appended, increment `ne` accordingly. Note: Hessian addition requires gradient data to be present; otherwise, an error is returned [S2, unofficial].
3.  Verify that all input MetaVolumes have the same number of sub-volumes (`numSubVols`) [S2, unofficial].
4.  Iterate through each sub-volume index `v` from `0` to `numSubVols - 1`:
    a. Retrieve dimensions `sx`, `sy`, `sz` [S2, unofficial].
    b. Allocate a contiguous data block `d` of size `sx * sy * sz * ne` [S2, unofficial].
    c. Initialize an offset pointer `offs` to 0 [S2, unofficial].
    d. For each voxel coordinate `(i, j, k)` where `i` is z, `j` is y, and `k` is x:
        i. Copy elements from the primary volume's current data `d0` into `d` starting at offset 0. The source index is `i*sx*sy*nelts + j*sx*nelts + k*nelts + e` and the destination is `i*sx*sy*ne + j*sx*ne + k*ne + e` for `e < nelts` [S2, unofficial].
        ii. Increment `offs` by `nelts` [S2, unofficial].
        iii. If additional MetaVolumes (`mv1`, `mv2`, `mv3`) are present, copy their data into `d` starting at the current `offs`, using their respective `nelts` and stride `ne` [S2, unofficial].
    e. Store the allocated block `d` as the output volume's `currentData` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def merge_channels(primary_mv, additional_mvs, add_g=False, add_h=False):
    """
    Merges multiple MetaVolumes into a single volume.
    primary_mv: The first MetaVolume object.
    additional_mvs: List of additional MetaVolume objects (up to 3).
    add_g: Boolean flag to append gradient magnitude.
    add_h: Boolean flag to append Hessian. Requires add_g=True.
    """
    if add_h and not add_g:
        raise ValueError("Hessian addition requires gradient data (add_g must be true)")

    # Calculate total elements per voxel (ne)
    ne = primary_mv.nelts
    if add_g:
        ne += 1
    if add_h:
        ne += 1

    # Verify sub-volume counts match
    num_sub_vols = primary_mv.numSubVols
    for mv in additional_mvs:
        if mv.numSubVols != num_sub_vols:
            raise ValueError("Input volumes must have the same number of sub-volumes")

    output_data = []

    for v in range(num_sub_vols):
        sx, sy, sz = primary_mv.get_dimensions(v)
        
        # Allocate output buffer for this sub-volume
        # Layout: [primary_nelts][mv1_nelts][mv2_nelts][mv3_nelts][gradient][hessian]
        # Stride is 'ne' per voxel
        d = [0] * (sx * sy * sz * ne)
        
        # Get source pointers
        d0 = primary_mv.get_current_data(v)
        sources = [d0]
        for mv in additional_mvs:
            sources.append(mv.get_current_data(v))
        
        # Append gradient/hessian sources if requested
        if add_g:
            sources.append(primary_mv.get_gradient_data(v)) # Assuming gradient is derived from primary or available
        if add_h:
            sources.append(primary_mv.get_hessian_data(v))

        # Copy data
        for i in range(sz): # z
            for j in range(sy): # y
                for k in range(sx): # x
                    dest_idx = (i * sy * sx + j * sx + k) * ne
                    source_offset = 0
                    
                    for src_data in sources:
                        src_nelts = src_data.nelts if hasattr(src_data, 'nelts') else (1 if src_data in [gradient, hessian] else primary_mv.nelts)
                        
                        # Copy elements
                        for e in range(src_nelts):
                            src_idx = (i * sy * sx + j * sx + k) * src_nelts + e
                            d[dest_idx + e] = src_data[src_idx]
                        
                        source_offset += src_nelts

        output_data.append(d)
    
    return output_data
```

**Parameters:**
*   `addG`: Boolean. If true, gradient magnitude is appended. Default: False [S2, unofficial].
*   `addH`: Boolean. If true, Hessian is appended. Default: False [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Bricked Volumes:** Gradient and Hessian computations are disabled for multi-subvolume (bricked) volumes. The implementation warns that gradients will be incorrect at brick boundaries and skips these calculations if `numSubVols != 1` [S2, unofficial].
*   **Dependency:** Hessian data cannot be added without gradient data [S2, unofficial].

## Subtopics

### MetaVolume Merging with Element Concatenation
A specific implementation of channel merging that combines up to four MetaVolumes into a single volume by concatenating per-voxel data elements (e.g., scalar + gradient + hessian) into a unified data layout [S2, unofficial]. It sets itself apart by explicitly supporting the concatenation of derived fields like gradients and Hessians alongside raw scalar data [S2, unofficial]. Choose this variant when you need to render multi-variable volumes that require derivative information for shading or feature enhancement, and your data is not bricked [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the total number of voxels and the number of channels being merged ($O(N \cdot C)$), where $N$ is the voxel count and $C$ is the number of channels [S2, unofficial].
*   **Memory Complexity:** Increases by a factor equal to the number of merged channels. If merging $C$ channels each with $E$ elements, the memory usage becomes $C \times E$ times the original single-channel memory [S2, unofficial].
*   **Strengths:** Simplifies data management for multi-modal rendering; allows simultaneous visualization of related variables (e.g., intensity and gradient) in a single pass [S1, unofficial].
*   **Limitations:** Not suitable for bricked volumes if gradient/hessian data is required due to boundary artifacts [S2, unofficial]. Large memory footprint for high-resolution multi-variable datasets [S1, unofficial].
*   **When to use:** When rendering multi-modal data where variables are spatially aligned and need to be processed together [S1, unofficial].
*   **When not to use:** When memory is constrained, or when working with bricked volumes that require accurate gradient information [S2, unofficial].

## Background
The concept was introduced in the context of the IEEE Visualization 2004 Contest, specifically within the "Cave" software suite [S1, unofficial]. It was developed to address the challenges of exploring multi-variable volume data sets where different modalities overlap in intensity space, making it difficult for users to interpret combined colors without extensive experience [S1, unofficial]. The method utilizes the `VConv` command-line tool from the Virvo software suite to perform the merging [S1, unofficial].

## Sources
[S1] IEEE Visualization 2004 Contest: User Friendly Volume Data Set Exploration in the Cave, http://web.eng.ucsd.edu/~jschulze/projects/vis04-contest/, fetched 2026-10-28 (unofficial)
[S2] ouyting/simian-spacemonkey, https://github.com/ouyting/simian-spacemonkey, fetched 2026-10-27 (unofficial)
