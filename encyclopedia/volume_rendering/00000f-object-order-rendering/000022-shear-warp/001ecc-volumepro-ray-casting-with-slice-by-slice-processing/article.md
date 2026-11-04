# VolumePro Ray-Casting with Slice-by-Slice Processing

Also called: VolumePro rendering, VoxelBlaster rendering

## Overview
VolumePro is a specialized hardware architecture that implements object-space ray-casting with parallel slice-by-slice processing to achieve real-time frame rates on PC-class computers [S1, unofficial]. It differs from texture-mapping approaches by supporting gradient estimation and per-sample Phong illumination in real-time, and from pre-processing approaches by allowing immediate visual feedback during parameter changes without lengthy recomputation [S1, unofficial]. It builds on the Shear-warp method [24-shear-warp.md] by casting rays from the base plane (the volume face most parallel to the viewing plane) into the dataset, projecting the resulting base plane image onto the final image plane via a 3D transformation and 2D warping [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Rectilinear isotropic or anisotropic volume data (8-bit or 12-bit scalar voxels), viewing parameters (parallel projection).
- Output: Final rendered image on the image plane.

**Data Structures:**
The architecture relies on specific FIFOs and shift registers to recirculate voxel data so it is available precisely when needed, as each voxel is read from memory exactly once per frame [S1, unofficial].
- **Voxel Memory:** Holds up to 128 MB of volume data. Accessed in slice-by-slice, beam-by-beam order [S1, unofficial].
- **Voxel Slice FIFO:** Stores input voxels to maintain neighborhood context for interpolation [S1, unofficial].
- **Voxel Beam FIFO:** Stores voxels along the beam direction [S1, unofficial].
- **Voxel Shift Register:** Stores individual voxels for immediate shift operations [S1, unofficial].
- **Sample Slice FIFO / Sample Beam FIFO / Sample Shift Register:** Buffers for interpolated sample values and gradients, sized to hold two full slices, two full beams, and two extra samples to handle gradient estimation latency [S1, unofficial].

**Pipeline Structure:**
The vg500 rendering chip contains four parallel rendering pipelines [S1, unofficial]. Each pipeline accepts a new voxel every cycle and forwards intermediate results every cycle without pipeline delays [S1, unofficial].

**Steps:**

1.  **Ray Casting and Base Plane Projection:**
    Rays are cast into the dataset from each pixel on the base plane (the face of the volume most parallel to the viewing plane) rather than in image space [S1, unofficial]. The resulting base plane image is projected onto the final image plane using a 3D transformation and a 2D image re-sampling operation (warp) [S1, unofficial]. VolumePro uses 2D texture mapping on a companion graphics card for this final warp [S1, unofficial].

2.  **Slice-by-Slice Traversal:**
    Voxels are read from memory in planes (slices) parallel to the base plane. Within a slice, voxels are read as a scanline (voxel beam) in top-to-bottom order [S1, unofficial]. This ensures regular, object-order data access [S1, unofficial].

3.  **Interpolation:**
    Each sample value is derived from its eight nearest neighboring voxels using trilinear interpolation [S1, unofficial].
    -   **Weights:** Trilinear weights $w_x, w_y, w_z$ are generated using a Digital Differential Analyzer (DDA) algorithm [S1, unofficial].
    -   **Weight Generation:** A single set of weights is sufficient for all samples of a slice because rays are parallel and spaced identically to pixel positions on the base plane [S1, unofficial]. The Weight Generator computes a new set of weights prior to the beginning of each slice, based on the ray angle to the base plane and the initial offset between rays and voxels on the first slice [S1, unofficial].
    -   **Precision:** Interpolated sample values are rounded to 12-bit precision [S1, unofficial]. Internal datapaths are 12-bits wide [S1, unofficial].

4.  **Gradient Estimation:**
    Gradients are estimated using a central difference gradient filter [S1, unofficial].
    -   **Hybrid Approach:** The estimation is divided. Z-gradients are estimated from voxel values and then interpolated to sample points. X- and Y-gradients are estimated from the interpolated sample points themselves [S1, unofficial].
    -   **Latency Handling:** Because the gradient at a sample point depends on the previous, current, and next slices, and the next slice is not yet available when the current sample arrives, the Gradient Estimation unit operates one voxel, one beam, and one slice behind the Interpolation unit [S1, unofficial]. When sample $S(x+1, y+1, z+1)$ arrives, the unit completes the gradient estimation for $S(x, y, z)$ [S1, unofficial].

5.  **Shading and Compositing:**
    The system supports per-sample Phong illumination and gradient magnitude modulation of opacity [S1, unofficial]. The pipeline composites samples as they are processed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumepro_render(volume, view_params, image_dims):
    # Initialize pipelines and FIFOs
    # Voxel Memory holds volume data
    # Four parallel pipelines process data
    
    # Pre-compute weights for the slice based on ray angle and initial offset
    # Using DDA algorithm [S1, unofficial]
    wx, wy, wz = generate_weights_dda(view_params.ray_angle, view_params.initial_offset)
    
    # Process slice by slice
    for slice_idx in range(volume.depth):
        # Process beam by beam within slice
        for beam_idx in range(volume.beam_length):
            # Read voxel from memory (once per frame) [S1, unofficial]
            voxel_stream = read_voxel_stream(volume, slice_idx, beam_idx)
            
            # Interpolation Unit
            # Uses Voxel Slice FIFO, Voxel Beam FIFO, Voxel Shift Register
            # to access 8 nearest neighbors [S1, unofficial]
            sample_stream = []
            for voxel in voxel_stream:
                # Trilinear interpolation using pre-computed weights
                sample = trilinear_interpolate(voxel, wx, wy, wz)
                # Round to 12-bit precision [S1, unofficial]
                sample = round_12bit(sample)
                sample_stream.append(sample)
            
            # Gradient Estimation Unit
            # Operates 1 voxel/beam/slice behind Interpolation [S1, unofficial]
            # Uses Sample Slice FIFO, Sample Beam FIFO, Sample Shift Register
            gradient_stream = []
            for i, sample in enumerate(sample_stream):
                # Z-gradient from voxel values, interpolated
                # X/Y gradients from interpolated samples
                # Central difference filter [S1, unofficial]
                gradient = estimate_gradient(sample_stream, voxel_stream, i)
                gradient_stream.append(gradient)
            
            # Shading and Compositing
            # Phong illumination and gradient magnitude modulation [S1, unofficial]
            for sample, gradient in zip(sample_stream, gradient_stream):
                color = phong_shade(sample, gradient)
                alpha = classify_opacity(sample, gradient)
                composite_into_base_plane(color, alpha)
    
    # Warp base plane image to final image plane
    # Using 2D texture mapping on companion graphics card [S1, unofficial]
    final_image = warp_image(base_plane_image, view_params.warp_matrix)
    return final_image
```

## Operating characteristics
-   **Performance:** Renders more than 500 million interpolated, Phong illuminated, composited samples per second [S1, unofficial].
-   **Capacity:** Sufficient to render a $256 \times 256 \times 256$ volume at 30 frames per second [S1, unofficial].
-   **Memory:** Supports up to 128 MB of volume memory [S1, unofficial].
-   **Strengths:** Real-time performance on PC hardware; supports gradient-based shading and opacity modulation; immediate visual feedback for parameter changes [S1, unofficial].
-   **Limitations:** Hardware-specific implementation (vg500 chip); currently supports only parallel projections [S1, unofficial].

## Background
VolumePro is based on the Cube-4 volume rendering architecture developed by SUNY at Stony Brook and Enhanced Memory Cube-4 (EM-Cube) [S1, unofficial]. It was developed by Hanspeter Pfister, Jan Hardenbergh, Jim Knittel, Hugh Lauer, and Larry Seiler at Mitsubishi Electric Research Laboratories (MERL) [S1, unofficial]. The system was described in a technical report (TR-99-19) in April 1999 and submitted to SIGGRAPH '99 [S1, unofficial]. It is the world's first single-chip real-time volume rendering system for PC class computers [S1, unofficial].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
