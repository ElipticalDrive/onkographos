# Integer-Stride Preview Subsampling

Also called: integer-stride sampling, nearest-neighbour voxel subsampling

## Overview
Integer-Stride Preview Subsampling generates low-resolution volume previews by subsampling the source volume using integer strides along each axis (Z, Y, X) [S1, unofficial]. It preserves exact original physical voxel positions by adjusting spacing while keeping origin and direction unchanged, making it suitable for navigation and rapid overview rather than diagnostic quality [S1, unofficial]. It differs from diagnostic resampling methods (like trilinear or cubic) by using nearest-neighbour subsampling, which is faster but not diagnostic-quality [S1, unofficial].

## How it works
**Inputs:** Source volume data (voxels), source geometry (shape, spacing, origin, direction), and a target `max_dimension` parameter.
**Outputs:** A `PreparedVolume` object containing the subsampled voxel array and updated geometry.

**Data Structures:**
*   `PreparedVolume`:
    *   `voxels`: numpy array, `float32`, C-contiguous, shape `(Z, Y, X)`.
    *   `geometry`: `VolumeGeometry` object containing:
        *   `shape_zyx`: tuple of integers.
        *   `spacing_xyz_mm`: tuple of floats.
        *   `origin_lps`: tuple of floats.
        *   `direction_lps`: tuple of floats (or matrix).
    *   `level`: integer (pyramid level index).
    *   `is_preview`: boolean.
    *   `source_shape_zyx`: tuple (original source dimensions).
    *   `stride_zyx`: tuple (subsampling strides).

**Steps:**
1.  **Calculate Strides:** For each axis $a \in \{Z, Y, X\}$, calculate the stride $s_a$ based on the source size $N_a$ and the target `max_dimension` $M$:
    $$ s_a = \max\left(1, \left\lceil \frac{N_a}{M} \right\rceil \right) $$
    [S1, unofficial]

2.  **Calculate New Shape:** Compute the dimensions of the preview volume using integer division:
    $$ N'_a = \left\lfloor \frac{N_a - 1}{s_a} \right\rfloor + 1 $$
    [S1, unofficial]

3.  **Extract Voxels:** Extract the subsampled voxels using slice notation with the calculated strides:
    $$ V_{preview} = V_{source}[::s_z, ::s_y, ::s_x] $$
    [S1, unofficial]

4.  **Ensure Contiguity:** Convert the result to a contiguous array in memory:
    $$ V_{contig} = \text{np.ascontiguousarray}(V_{preview}) $$
    [S1, unofficial]

5.  **Handle Memory Sharing:** If the contiguous array shares memory with the source (which occurs if all strides are 1), create a deep copy to ensure separation:
    ```python
    if np.shares_memory(V_contig, V_source):
        V_final = V_contig.copy()
    else:
        V_final = V_contig
    ```
    [S1, unofficial]

6.  **Mark Write-Only:** Mark the preview pixel buffer as write-only (read-only for the application) to prevent accidental modification:
    ```python
    V_final.setflags(write=False)
    ```
    [S1, unofficial]

7.  **Update Geometry:** Keep the original `origin_lps` and `direction_lps`. Multiply the original spacing by the respective strides to maintain the physical bounds:
    $$ \text{spacing}'_a = \text{spacing}_a \times s_a $$
    [S1, unofficial]

**Parameters:**
*   `max_dimension`: The maximum allowed dimension size for any axis of the preview volume. Typical values are derived from `DEFAULT_PREVIEW_DIMENSIONS` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Stride 1:** If the stride is 1 in every direction, NumPy's slicing returns a view that shares memory with the original buffer. This must be detected and handled by forcing a copy to avoid unintended side effects on the source data [S1, unofficial].
*   **Precision:** The method uses nearest-neighbour sampling, which is fast but does not preserve intensity averages, making it unsuitable for diagnostic tasks where intensity accuracy is critical [S1, unofficial].

## Subtopics

### Preview Pyramid Generation
A method to generate a hierarchy of low-resolution volume levels (a pyramid) from a full-resolution volume [S1, unofficial]. It creates independent contiguous `float32` previews for various `max_dimension` limits [S1, unofficial]. The process retrieves the preview configuration (enabled flag and `max_dimension` limit) from settings [S1, unofficial]. It determines the list of target dimensions by intersecting `DEFAULT_PREVIEW_DIMENSIONS` with the configured `max_dimension`, then sorting and deduplicating them [S1, unofficial]. It iterates through each limit, calculating stride and shape, skipping generation if the shape matches the full-resolution volume or has already been generated (deduplication via a `seen_shapes` set) [S1, unofficial]. For each valid limit, it calls the core subsampling method with an incrementing level counter and appends the result to a `VolumePackage` which contains the full resolution volume and a tuple of preview `PreparedVolume` objects [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the source volume, as it involves a single pass to extract elements and potentially copy them.
*   **Memory Complexity:** $O(N')$ where $N'$ is the number of voxels in the preview volume. If a copy is forced, peak memory usage is $O(N + N')$.
*   **Performance:** Extremely fast due to simple indexing and lack of interpolation. Suitable for real-time navigation.
*   **Strengths:** Preserves physical position (origin/direction) while reducing data size; very low computational overhead.
*   **Limitations:** Not diagnostic quality; loses intensity information (nearest-neighbour); may produce aliasing artifacts if strides are large.
*   **When to use:** Rapid overview, navigation, thumbnail generation.
*   **When not to use:** Diagnostic analysis, quantitative measurements requiring intensity accuracy.

## Background
Originating from the implementation details of the Sinus 3D Viewer Pro project, this method is part of a broader strategy for handling large medical volumes efficiently [S1, unofficial]. It is designed to support user interfaces that require quick access to volume data without the latency of full-resolution rendering or complex resampling.

## Sources
[S1] sarfrazkhan66003/sinus-3d-viewer-pro, https://github.com/sarfrazkhan66003/sinus-3d-viewer-pro, fetched 2026-10-26 (unofficial)
