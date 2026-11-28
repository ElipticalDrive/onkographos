# GPU Volume Slicing (Canonical)

## Overview
GPU Volume Slicing (Canonical) is a GPU kernel that renders a single 2D slice (coronal, sagittal, or horizontal) from a 3D volume dataset by mapping screen coordinates to volume coordinates using canonical orientations and applying translation and scaling [S1, unofficial]. It builds on [2D Volume Cross-Section Rendering](1764-2d-volume-cross-section-rendering.md) by providing the specific coordinate transformation logic for orthographic cross-sections. It differs from [Cross Section Filter](15347-cross-section-filter.md) by operating as a direct pixel shader/kernel rather than a post-processing filter, and from [Orthographic Crosshair Projection](15605-orthographic-crosshair-projection.md) by focusing on single-plane extraction rather than multi-planar reconstruction overlays.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D texture (`texVolume`) containing the volume data; screen dimensions (`bp.width`, `bp.height`); slice parameters (`sp.dx`, `sp.dy`, `sp.dz`); scale factor (`scale`); canonical orientation (`canonicalOrientation c`).
*   **Output:** A 2D buffer (`buffer`) containing the sampled slice values.

**Data Structures:**
*   `bp`: A struct or object containing `width` and `height` of the output buffer.
*   `sp`: A struct or object containing `dx`, `dy`, `dz` translation offsets.
*   `canonicalOrientation`: An enum with values `SAGITTAL`, `HORIZONTAL`, `CORONAL`.

**Numbered Steps:**
1.  **Launch Configuration:** The kernel is launched with a grid of `dim3(dim/16, dim/16)` blocks and `dim3(16, 16)` threads per block, where `dim` is typically 1024 [S1, unofficial].
2.  **Thread Indexing:** Each thread calculates its pixel coordinates `i` (x) and `j` (y) based on block and thread indices. It computes a linear offset in the output buffer: `offset = j * bp.height + i` [S1, unofficial].
3.  **Boundary Check:** If `j >= bp.height` or `i >= bp.width`, the thread exits without writing [S1, unofficial].
4.  **Coordinate Normalization:** Initialize a position vector `pos` to `(0, 0, 0)`. Map screen coordinates to the normalized range `[0, 1]`:
    *   `pos.x += (float)i / (float)bp.width`
    *   `pos.y += (float)j / (float)bp.height` [S1, unofficial].
5.  **Orientation Mapping:** Set the third coordinate (`pos.z`) based on the canonical orientation `c`:
    *   If `SAGITTAL`: `pos.z += 0`
    *   If `HORIZONTAL`: `pos.z += (float)i / (float)bp.width`
    *   If `CORONAL`: `pos.z += (float)i / (float)bp.width` [S1, unofficial].
6.  **Translation:** Add slice parameters to the position:
    *   `pos.x += sp.dx`
    *   `pos.y += sp.dy`
    *   `pos.z += sp.dz` [S1, unofficial].
7.  **Scaling and Centering:** Transform the position to volume texture coordinates using the formula:
    $$pos = \frac{pos - 0.5}{scale} + 0.5$$ [S1, unofficial].
8.  **Texture Sampling:** Perform a bounds check on `pos`. If `pos` is within `[0, 1]` for all components, sample the 3D texture: `sample = tex3D(texVolume, pos.x, pos.y, pos.z)`. If invalid, set `sample = 0.0f` [S1, unofficial].
9.  **Output:** Write the sampled value to the buffer: `buffer[offset] = sample` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slice_kernel(texVolume, buffer, bp, sp, scale, canonicalOrientation):
    # Calculate thread indices
    block_idx = (blockIdx.x, blockIdx.y)
    thread_idx = (threadIdx.x, threadIdx.y)
    
    # Global pixel coordinates
    i = block_idx.x * 16 + thread_idx.x
    j = block_idx.y * 16 + thread_idx.y
    
    # Boundary check
    if j >= bp.height or i >= bp.width:
        return

    # Linear offset
    offset = j * bp.height + i
    
    # Coordinate normalization
    pos_x = (float(i) / float(bp.width))
    pos_y = (float(j) / float(bp.height))
    pos_z = 0.0
    
    # Orientation mapping
    if canonicalOrientation == SAGITTAL:
        pos_z += 0.0
    elif canonicalOrientation == HORIZONTAL:
        pos_z += (float(i) / float(bp.width))
    elif canonicalOrientation == CORONAL:
        pos_z += (float(i) / float(bp.width))
        
    # Translation
    pos_x += sp.dx
    pos_y += sp.dy
    pos_z += sp.dz
    
    # Scaling and centering
    pos_x = (pos_x - 0.5) / scale + 0.5
    pos_y = (pos_y - 0.5) / scale + 0.5
    pos_z = (pos_z - 0.5) / scale + 0.5
    
    # Bounds check and sampling
    if 0.0 <= pos_x <= 1.0 and 0.0 <= pos_y <= 1.0 and 0.0 <= pos_z <= 1.0:
        sample = tex3D(texVolume, pos_x, pos_y, pos_z)
    else:
        sample = 0.0
        
    buffer[offset] = sample
```

**Parameters:**
*   `dim`: Grid dimension base, typically 1024 [S1, unofficial].
*   `scale`: Scaling factor for volume coordinates. Not given in the sources.
*   `sp.dx, sp.dy, sp.dz`: Translation offsets. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point division and subtraction may introduce precision errors near boundaries.
*   **Boundaries:** Threads outside the buffer dimensions must exit early to prevent out-of-bounds writes [S1, unofficial].
*   **Invalid Coordinates:** If `pos` falls outside `[0, 1]` after scaling, the sample is clamped to 0.0f [S1, unofficial].

## Subtopics

### Cross Section Filter
Applies filtering to cross-sections. Use when post-processing is needed. [Cross Section Filter](15347-cross-section-filter.md)

### Orthographic Crosshair Projection
Projects crosshairs onto orthographic views. Use for multi-planar overlays. [Orthographic Crosshair Projection](15605-orthographic-crosshair-projection.md)

### Slice Viewport Scaling and Panning
Handles viewport transformations. Use for interactive navigation. [Slice Viewport Scaling and Panning](21023-slice-viewport-scaling-and-panning.md)

### Curved Plane Reconstruction
Reconstructs curved planes. Use for non-planar slices. [Curved Plane Reconstruction](22024-curved-plane-reconstruction.md)

## Operating characteristics
*   **Time Complexity:** O(N) where N is the number of pixels in the output buffer.
*   **Memory Complexity:** O(N) for the output buffer.
*   **Performance:** Depends on GPU hardware and resolution. Typical resolution is 1024x1024 [S1, unofficial].
*   **Strengths:** Simple implementation, efficient for single-slice rendering.
*   **Limitations:** Limited to planar slices; performance degrades with very high resolutions if not optimized.
*   **When to use:** Interactive single-slice visualization.
*   **When not to use:** Multi-planar reconstruction or volume rendering.

## Background
Originated in the `volume-viz` project by Jacob Stern [S1, unofficial]. It is used in medical imaging and scientific visualization for interactive slice viewing.

## Sources
[S1] jacobstern/volume-viz, https://github.com/jacobstern/volume-viz, fetched 2026-10-27 (unofficial)
