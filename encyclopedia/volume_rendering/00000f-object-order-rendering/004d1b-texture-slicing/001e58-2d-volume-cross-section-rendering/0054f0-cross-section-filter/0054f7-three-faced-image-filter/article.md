# Three-faced Image Filter

Also called: Mid-plane Slicing, Orthogonal Slice Display, Orthogonal Slice Visualization, Slice Extraction

## Overview
The Three-faced Image Filter extracts and renders three orthogonal 2D cross-sections (xy, xz, and yz planes) from a 3D scalar volume array simultaneously [S1, unofficial][S3]. It solves the problem of visualizing internal volume structure without the computational cost of full 3D raycasting, providing an immediate "sagittal, coronal, and axial" view [S1, unofficial][S3]. Unlike single-slice filters, it displays all three principal planes at once, allowing for immediate spatial correlation across axes [S3]. It differs from general cross-section filters by enforcing orthogonality and fixed mid-plane extraction rather than arbitrary user-defined planes [S1, unofficial][S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume represented as a 1D array of length $N^3$, where values are in the range $[0, 255]$ [S1, unofficial].
*   **Outputs:** Three 2D images (RGBA) corresponding to the xy, xz, and yz planes [S1, S2, unofficial].

**Data Structures:**
*   **Volume Array:** 1D array `u` of size $N \times N \times N$ [S1, unofficial].
*   **Slice Planes:** Three 2D grids of size $N \times N$ [S1, unofficial].
*   **Parameters:**
    *   `size`: The dimension $N$ of the cubic volume [S1, unofficial].
    *   `mid`: The fixed index for the cross-section, calculated as `floor(size / 2)` [S1, unofficial].
    *   `colorLevel`: Brightness/center of the display window [S2, unofficial].
    *   `colorWindow`: Contrast/width of the display window [S2, unofficial].
    *   `offset`: Distance between the projected image and the volume [S3].

**Steps:**
1.  **Initialize:** Determine `size` and `mid` [S1, unofficial].
2.  **Iterate Planes:** For each of the three orthogonal planes (xy, xz, yz):
    a.  Iterate $a$ from $0$ to $size-1$ and $b$ from $0$ to $size-1$ [S1, unofficial].
    b.  Compute the 1D index in the volume array based on the plane type:
        *   **xy plane:** $index = a \cdot size^2 + b \cdot size + mid$ [S1, unofficial].
        *   **xz plane:** $index = a \cdot size^2 + mid \cdot size + b$ [S1, unofficial].
        *   **yz plane:** $index = mid \cdot size^2 + a \cdot size + b$ [S1, unofficial].
    c.  Retrieve the scalar value $v = u[index]$ [S1, unofficial].
    d.  Normalize the value to $[0, 1]$ by computing $v_{norm} = v / 255$ [S1, unofficial].
    e.  Apply the color ramp/transfer function to $v_{norm}$ to determine RGBA components [S1, S2, unofficial].
    f.  Write the RGBA data to the output image buffer at pixel offset $(b \cdot size + a) \cdot 4$ [S1, unofficial].
3.  **Rendering Setup:** Instantiate three slice actors (e.g., `vtkImageSlice`) [S2, unofficial].
4.  **Apply Window/Level:** Set `colorLevel` and `colorWindow` properties on all three actors to control brightness and contrast [S2, unofficial].
5.  **Render:** Trigger a render event to display the slices [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def three_faced_image_filter(volume_array, size, color_level, color_window, color_ramp):
    """
    Extracts and renders three orthogonal mid-plane slices.
    
    Args:
        volume_array: 1D list/array of length size^3, values in [0, 255].
        size: Integer dimension of the cubic volume.
        color_level: Float, brightness center.
        color_window: Float, contrast width.
        color_ramp: Function mapping normalized scalar [0,1] to RGBA tuple.
        
    Returns:
        dict: Three 2D RGBA arrays for 'xy', 'xz', 'yz' planes.
    """
    mid = size // 2
    slices = {'xy': None, 'xz': None, 'yz': None}
    
    # Pre-allocate output buffers
    for plane in slices:
        slices[plane] = [[(0,0,0,0) for _ in range(size)] for _ in range(size)]
        
    for a in range(size):
        for b in range(size):
            # Calculate 1D index for each plane
            idx_xy = a * (size ** 2) + b * size + mid
            idx_xz = a * (size ** 2) + mid * size + b
            idx_yz = mid * (size ** 2) + a * size + b
            
            # Retrieve scalar value
            val_xy = volume_array[idx_xy]
            val_xz = volume_array[idx_xz]
            val_yz = volume_array[idx_yz]
            
            # Normalize to [0, 1]
            norm_xy = val_xy / 255.0
            norm_xz = val_xz / 255.0
            norm_yz = val_yz / 255.0
            
            # Apply color ramp (simplified; real implementation uses colorLevel/Window)
            rgba_xy = color_ramp(norm_xy)
            rgba_xz = color_ramp(norm_xz)
            rgba_yz = color_ramp(norm_yz)
            
            # Write to 2D slice buffer
            slices['xy'][a][b] = rgba_xy
            slices['xz'][a][b] = rgba_xz
            slices['yz'][a][b] = rgba_yz
            
    return slices
```

**Edge Cases and Pitfalls:**
*   **Interpolation:** Canvas scaling must disable interpolation to preserve sharp voxel boundaries. Set `imageSmoothingEnabled = false` before drawing [S1, unofficial].
*   **Boundary Conditions:** The mid-plane index is fixed at `floor(size / 2)`. If `size` is even, this selects the lower-middle plane [S1, unofficial].
*   **Color Mapping:** The color level and window must be applied consistently across all three slices to maintain visual coherence [S2, unofficial].

## Subtopics

### Single-Slice Filter
Displays only one arbitrary cross-section. Choose when memory is constrained or when focusing on a specific anatomical plane. [Cross Section Filter](15347-cross-section-filter.md)

### Multi-Planar Reconstruction (MPR)
Allows interactive sliding of slice planes. Choose when precise localization within the volume is required rather than a fixed mid-view. [Cross Section Filter](15347-cross-section-filter.md)

## Operating characteristics
*   **Time Complexity:** $O(N^2)$ per plane, $O(N^2)$ total for three planes, where $N$ is the volume dimension. This is significantly faster than full volume rendering ($O(N^3)$).
*   **Memory Complexity:** $O(N^2)$ for the output images. The input volume is $O(N^3)$.
*   **Performance:** Provides real-time visualization on standard hardware for moderate resolutions ($N \le 256$). Performance degrades linearly with resolution.
*   **Strengths:** Immediate visualization of internal structures; low computational cost; simple implementation.
*   **Limitations:** Only shows three fixed planes; misses structures not intersecting the mid-plane; no depth perception within the slice.
*   **When to Use:** Quick inspection of volume data; debugging volume generation; applications where full 3D rendering is too slow.
*   **When Not to Use:** When detailed internal structure outside the mid-plane is critical; when depth cues are necessary.

## Background
The Three-faced Image Filter is a standard technique in medical imaging and scientific visualization for displaying orthogonal views of volumetric data [S1, unofficial][S3]. It is commonly implemented in visualization toolkits like VTK and vtk.js [S2, unofficial]. The method is often referred to as "mid-plane slicing" or "orthogonal slice display" [S1, S2, unofficial]. It is particularly useful for DICOM image viewers and scientific data explorers [S2, unofficial][S3].

## Sources
[S1] dueyama/gray-scott-3d, https://github.com/dueyama/gray-scott-3d, fetched 2026-10-27 (unofficial)
[S2] abbas695/DICOM-image-viewer, https://github.com/abbas695/DICOM-image-viewer, fetched 2026-10-27 (unofficial)
[S3] DF3 Viewer Oosawa, https://qcganime.web.fc2.com/OOSAWA/MANUAL/FilterE.html, fetched 2026-10-29
