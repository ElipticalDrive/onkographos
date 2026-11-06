# Cube-3 Hybrid Ray-Casting

Also called: Cube-3, Cube-3 Architecture, Cubic Frame Buffer rendering

## Overview
Cube-3 is a highly parallel, pipelined hardware architecture and hybrid ray-casting algorithm designed for the real-time visualization of high-resolution volumetric datasets (e.g., $512^3$ voxels) [S1]. It solves the traditional memory access bottleneck in volume rendering by ensuring at most one memory access per voxel per projection without pre-processing or data-dependent optimizations [S1]. Unlike object-order methods like VIRIM or image-order methods like VOGUE, which process single ray samples per pipeline stage, Cube-3 processes all sample values belonging to a ray simultaneously, exploiting spatial coherency to achieve the largest possible degree of parallelism [S1].

## How it works
The method combines discrete ray tracing with hybrid interpolation to generate continuous ray samples from a skewed memory structure, shading them, and compositing them via a hierarchical tree.

### Inputs and Outputs
*   **Input:** A volumetric dataset of size $n^3$ voxels, where each voxel is 16 bits. The viewing direction defines a "base-plane" (the volume face most perpendicular to the view).
*   **Output:** A projected image on the base-plane, which is subsequently warped and interpolated by the host into the final view plane image [S1].

### Data Structures and Layout
1.  **Cubic Frame Buffer (CFB):** The dataset is stored in $n$ dual-access memory modules. Each module stores $n^2$ voxels.
    *   **Mapping:** A voxel at coordinates $(x, y, z)$ is mapped to module $k$ using:
        $$k = (x + y + z) \bmod n$$
        where $0 \leq k, x, y, z \leq n-1$ [S1].
    *   **Internal Layout:** Within module $k$, the voxel is stored at indices $(i, j)$ where $i=x$ and $j=y$. This skewed organization ensures that any beam of $n$ voxels parallel to a primary axis resides in different memory modules, preventing access conflicts [S1].
2.  **Discrete Ray Buffers:** Part of the TRILIN units, these store beams fetched from the CFB.
3.  **ABC Buffers:** Three buffers storing continuous planes of ray samples: **A**bove, **B**elow, and **C**urrent. These are used for gradient estimation and shading [S1].
4.  **Ray Projection Cone (RPC):** A folded and circular binary tree of $n-1$ Voxel Combination Units (VCUs) with $n$ shading units as leaves [S1].

### Algorithm Steps

1.  **Generate Discrete Rays:**
    *   Cast discrete voxel-rays from each pixel on the base-plane.
    *   Use a 3D variation of Bresenham's line drawing algorithm to generate these rays from continuous viewing rays [S1].
    *   This guarantees constant stepping of distance 1 along the major axis (e.g., Z).
    *   Steps in non-major directions (e.g., X, Y) are determined by lookup tables called **templates**. These templates ensure that neighboring discrete rays do not overlap in voxels [S1].
    *   Discrete rays belonging to a scanline form a **Projection Ray Plane (PRP)**.

2.  **Fetch Beams:**
    *   Fetch PRPs as a sequence of voxel beams from the CFB according to the discrete ray templates [S1].
    *   Beams are stored in the discrete ray buffers of the TRILIN units.

3.  **Align and Store:**
    *   Use a high-bandwidth global bus (**Fast Bus**) to align all discrete rays in each PRP parallel to a main axis of the 2D buffers [S1].
    *   Use 2D skewing (similar to CFB) to support conflict-free storage of beams from the CFB and conflict-free retrieval of axis-aligned discrete rays [S1].

4.  **Interpolate (TRI-LIN):**
    *   Fetch consecutive discrete rays each clock cycle into $n$ tri-linear interpolation units (TRI-LINs) [S1].
    *   **Sheared Tri-linear Interpolation:** Two discrete PRPs (top and bottom) are used to generate sample values for the original continuous rays. Interpolation uses voxels from neighboring discrete rays of the top and bottom PRPs [S1].
    *   **Gradient Estimation:** Estimate the gradient at each resample location using central differences between samples of rays on the immediate left, right, above, below, and along the current ray [S1].
    *   **Sharing:** Overlapping voxels of neighboring discrete rays are shared between neighboring TRILIN units. This allows sharing already computed ray samples between neighboring rays, avoiding additional CFB accesses [S1].
    *   Store the interpolated continuous sample values and gradients in the ABC buffers [S1].

5.  **Shade (Shaders):**
    *   Forward interpolated sample and gradient values to shading units (Shaders) [S1].
    *   **Opacity:** Assign opacity values to each sample using a user-controlled look-up table (transfer function) [S1].
    *   **Color:** Calculate color value at each sample location according to a local illumination model [S1].

6.  **Composite (RPC):**
    *   Connect the $n$ shading units as leaves of the Ray Projection Cone (RPC), a folded and circular binary tree containing $n-1$ Voxel Combination Units (VCUs) [S1].
    *   The RPC takes one ray of opacity and color samples and generates one projected pixel value per clock cycle [S1].
    *   **Projection Schemes:** The RPC implements schemes such as first/last opaque projection, maximum voxel value, weighted summation, or standard compositing [S1].

7.  **Output:**
    *   Transmit the resulting pixel of the base-plane to the host.
    *   The host warps and stores the result in the frame-buffer [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Cube-3 Hybrid Ray-Casting Implementation Logic
# Note: This is a logical representation of the hardware pipeline described in [S1].

def cube3_render(dataset, n, viewing_direction, transfer_function, illumination_model):
    """
    dataset: 3D array of size n^3, 16-bit voxels
    n: dimension size
    viewing_direction: vector defining the view
    transfer_function: LUT mapping voxel values to opacity
    illumination_model: function for local lighting
    """
    
    # 1. Setup CFB Mapping
    # k = (x + y + z) % n
    # Internal index: (i, j) = (x, y)
    
    # 2. Generate Discrete Rays from Base-Plane
    # Base-plane is the volume face most perpendicular to viewing_direction
    base_plane_pixels = get_base_plane_pixels(viewing_direction, n)
    
    # Templates for non-major axis steps (Bresenham 3D variation)
    templates = generate_bresenham_templates(viewing_direction, n)
    
    discrete_rays = []
    for pixel in base_plane_pixels:
        ray = generate_discrete_ray(pixel, viewing_direction, templates, n)
        discrete_rays.append(ray)
        
    # Group rays into Projection Ray Planes (PRPs) by scanline
    prps = group_rays_into_prps(discrete_rays)
    
    # 3. Fetch and Align Beams
    # Fetch beams from CFB. Conflict-free due to skewed mapping.
    # Align rays on Fast Bus to match 2D buffer axis.
    aligned_prps = []
    for prp in prps:
        beams = fetch_beams_from_cfb(prp, n)
        aligned_prp = align_beams_to_axis(beams, viewing_direction)
        aligned_prps.append(aligned_prp)
        
    # 4. Interpolation and Gradient Estimation (TRI-LIN)
    # Use top and bottom PRPs for sheared tri-linear interpolation
    # Store in ABC buffers (Above, Below, Current)
    abc_buffers = initialize_abc_buffers(n)
    
    for i in range(len(aligned_prps) - 1):
        top_prp = aligned_prps[i]
        bottom_prp = aligned_prps[i+1]
        
        # Interpolate continuous samples from discrete beams
        # Share overlapping voxels between neighboring TRILIN units
        continuous_samples = sheared_trilinear_interpolate(top_prp, bottom_prp, n)
        
        # Estimate gradients using central differences from neighbors
        # Neighbors: left, right, above, below, along current ray
        gradients = estimate_gradients(continuous_samples, abc_buffers, n)
        
        # Store in ABC buffers
        update_abc_buffers(continuous_samples, gradients, abc_buffers)
        
    # 5. Shading
    shaded_rays = []
    for ray_samples in continuous_samples: # Iterate over rays
        shaded_samples = []
        for sample in ray_samples:
            # Opacity from transfer function
            opacity = transfer_function(sample.value)
            
            # Color from illumination model using gradient
            color = illumination_model(sample.value, gradients[sample.location])
            
            shaded_samples.append({
                'opacity': opacity,
                'color': color,
                'distance': sample.distance
            })
        shaded_rays.append(shaded_samples)
        
    # 6. Composition via Ray Projection Cone (RPC)
    # RPC is a binary tree of VCU nodes
    # Leaves are shading units, internal nodes are VCUs
    final_pixels = []
    for ray_shaded_samples in shaded_rays:
        # Composite ray samples into one pixel value
        # Supports: first/last opaque, max voxel, weighted sum, compositing
        pixel_value = composite_ray(ray_shaded_samples, projection_scheme='compositing')
        final_pixels.append(pixel_value)
        
    # 7. Output
    # Transmit base-plane pixels to host for warping
    return final_pixels
```

### Parameters and Edge Cases
*   **Dataset Size:** Typically $n^3$ where $n$ is a power of 2 (e.g., 128, 256, 512) [S1].
*   **Voxel Depth:** 16 bits [S1].
*   **Memory Modules:** $n$ modules required [S1].
*   **Edge Cases:**
    *   **Memory Conflicts:** Avoided by the skewed CFB organization and 2D skewing in buffers. If $n$ is not chosen such that the modulo arithmetic distributes beams correctly, conflicts may arise [S1].
    *   **Ray Overlap:** The use of templates in Bresenham's algorithm specifically avoids overlap of voxels between neighboring discrete rays, which is critical for the sharing mechanism in TRILINs [S1].
    *   **Precision:** The method relies on fixed-point or specific hardware floating-point units for the interpolation and shading; precision limits are not explicitly detailed in the sources but are implied by the 16-bit input and hardware constraints [S1].

## Subtopics

### [Extended Cube Architecture for Arbitrary Parallel Projection](3375-extended-cube-architecture-for-arbitrary-parallel-projection.md)
Extends Cube-3 to support arbitrary parallel projections by using a 2D Skewed Buffer and Ray Projection Tree, overcoming the orthographic limitation of the original Cube.

### [Discrete Ray Tracing](3020-discrete-ray-tracing.md)
The parent method providing the foundation for generating discrete voxel-rays from continuous viewing rays.

## Operating characteristics
*   **Performance:** Limited only by the data-transfer rate on the Fast Bus due to fully pipelined implementation [S1].
    *   128^3 dataset at 8MHz bus: 30Hz [S1].
    *   256^3 dataset at 33MHz bus: 30Hz [S1].
    *   512^3 dataset at 66MHz bus: 15Hz [S1].
    *   512^3 dataset at 125MHz bus: 30Hz [S1] [S1].
*   **Hardware Requirements:** A Cube-3 implementation for 30 projections per second of a 512^3 16-bit dataset requires 8 custom boards and a specially fabricated bus backplane [S1].
*   **Complexity:** The architecture exhibits the largest possible degree of parallelism compared to VIRIM and VOGUE [S1].
*   **Strengths:** Real-time visualization of high-resolution datasets; no pre-processing required; solves memory access bottleneck; supports 4D visualization (dynamic datasets) [S1].
*   **Limitations:** Hardware-specific; requires high-bandwidth global bus (Fast Bus) which becomes a bottleneck for larger datasets or higher frame rates (leading to Cube-4 development) [S1].

## Background
Cube-3 was developed at the State University of New York at Stony Brook [S1]. It was designed to enable real-time 4D visualization of dynamically changing datasets, such as in-situ fluid flow in rocks [S1]. The architecture is a hybrid ray-casting algorithm that exploits coherency to solve the traditional memory access bottleneck, contrasting with object-order (VIRIM) and image-order (VOGUE) approaches [S1]. The work is documented in publications from 1995 [S1].

## Sources
[S1] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
