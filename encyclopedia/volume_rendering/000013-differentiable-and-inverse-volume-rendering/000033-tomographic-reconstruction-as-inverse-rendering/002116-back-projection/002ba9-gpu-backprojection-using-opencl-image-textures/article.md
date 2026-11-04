# GPU Backprojection using OpenCL Image Textures

Also called: GPU-Image Backprojection

## Overview
This method accelerates the backprojection step of the Feldkamp-Davis-Kress (FDK) algorithm by leveraging OpenCL image objects (`image2d_t`) for hardware-accelerated interpolation of pre-filtered projection data. It differs from the buffer-based sibling methods by offloading bilinear interpolation to the GPU's texture cache, which typically yields higher performance for large projection datasets. It builds on [Back-projection](2450-back-projection.md) and assumes pre-filtering has been performed on the CPU.

## How it works
The method processes each projection angle sequentially. For each angle, the raw projection data is transposed, loaded into an OpenCL image object, and used to backproject into the volume buffer.

### Inputs and Outputs
**Inputs:**
- `projections`: `std::vector<float>` containing raw projection data.
- `P`: Number of projections.
- `W`, `H`: Detector width and height in pixels.
- `NX`, `NY`: Volume dimensions. The volume is assumed to be cubic in X/Y, i.e., $NX \times NX \times NY$.
- `voxelSize`: Physical size of a voxel (mm).
- `pixelSize`: Physical size of a detector pixel (mm).
- `sdd`: Source-to-Detector Distance (mm).
- `sod`: Source-to-Object Distance (mm).

**Outputs:**
- `volume`: `std::vector<float>` representing the reconstructed 3D volume, layout $[0..NX-1, 0..NX-1, 0..NY-1]$.

### Data Structures
**OpenCL Image2D:**
- Channel Order: `CL_R`
- Channel Data Type: `CL_FLOAT`
- Image Type: `CL_MEM_OBJECT_IMAGE2D`
- Width: `W`
- Height: `H`
- Row Pitch: `0` (native alignment)

**Kernel Arguments:**
1. `cl_mem image`: The projection image (`CL_MEM_READ_ONLY`).
2. `cl_mem volumeBuffer`: Output volume buffer.
3. `int W`, `int H`: Detector dimensions.
4. `int NX`, `int NY`: Volume dimensions.
5. `float voxelSize`, `float pixelSize`: Physical scales.
6. `float sdd`, `float sod`: Geometry distances.
7. `float angle`: Current projection angle in radians.

### Steps
1. **Pre-filtering (CPU):** Apply FDK pre-filtering (cone weighting and ramp filter) to the raw projections on the CPU. The source notes specify this step occurs before GPU processing.
2. **Data Transposition:** For each projection $p$, convert the data from column-major (`psrc[u * H + v]`) to row-major (`dest[v * W + u]`) to match the image texture layout.
3. **Loop over Projections:** Iterate $p$ from $0$ to $P-1$.
    a. Calculate angle: $\theta = \frac{2\pi p}{P}$.
    b. **Image Creation:** Create an OpenCL `image2d_t` from the transposed row-major data with flags `CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR`.
    c. **Kernel Execution:** Enqueue the backprojection NDRange kernel.
        - Global Size: $NX \times NX \times NY$.
        - Each work-item $(x, y, z)$ computes the contribution of the current projection to that voxel.
    d. **Cleanup:** Release the image memory object.
4. **Final Scaling:** After reading the volume buffer back to the host, multiply every voxel by $\frac{\pi}{P}$.

### Formulas
For each voxel $(x, y, z)$ and projection angle $\theta$:

$$
t = y \cos(\theta) - x \sin(\theta)
$$

$$
U = \text{sod} + y \sin(\theta) + x \cos(\theta)
$$

$$
u_{\text{proj}} = \text{sdd} \cdot \frac{t}{U}
$$

$$
v_{\text{proj}} = \text{sdd} \cdot \frac{z}{U}
$$

The contribution to the voxel is:

$$
\text{volume}[x, y, z] \ += \ \text{filtered\_proj}(u_{\text{proj}}, v_{\text{proj}}) \cdot \frac{\text{sod}^2}{U^2}
$$

Where $\text{filtered\_proj}(u, v)$ is obtained via hardware bilinear interpolation from the OpenCL image object.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gpu_backprojection_image(projections, P, W, H, NX, NY, voxelSize, pixelSize, sdd, sod):
    """
    Simulates the logic of the GPU Image Backprojection method.
    Note: Actual OpenCL calls are abstracted; this shows the data flow and math.
    """
    volume = [0.0] * (NX * NX * NY)
    
    # Pre-filtering is assumed done on CPU before this function call
    # projections are now filtered
    
    for p in range(P):
        # 1. Transpose projection data from column-major to row-major
        # Original: psrc[u * H + v]
        # Target:   rowMajor[v * W + u]
        proj_row_major = [0.0] * (W * H)
        for u in range(W):
            for v in range(H):
                src_idx = u * H + v
                dst_idx = v * W + u
                proj_row_major[dst_idx] = projections[p][src_idx]
        
        # 2. Create OpenCL Image2D (Abstracted)
        # image = clCreateImage2D(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR, ...)
        # image_data = proj_row_major
        
        # 3. Calculate Angle
        angle = 2.0 * math.pi * p / P
        
        # 4. Kernel Execution: Backproject one projection
        # Global size: NX * NX * NY
        # Each work-item (x, y, z) runs the following logic:
        
        for x in range(NX):
            for y in range(NX):
                for z in range(NY):
                    # Compute detector coordinates
                    t = y * math.cos(angle) - x * math.sin(angle)
                    U = sod + y * math.sin(angle) + x * math.cos(angle)
                    
                    # Avoid division by zero
                    if U == 0:
                        continue
                        
                    u_proj = sdd * t / U
                    v_proj = sdd * z / U
                    
                    # Hardware interpolation of filtered projection
                    # In OpenCL: read_imagef(image, sampler, (float2)(u_proj, v_proj))
                    # We simulate bilinear interpolation here for completeness
                    val = hardware_interpolate(proj_row_major, u_proj, v_proj, W, H)
                    
                    # Accumulate with weighting
                    weight = (sod * sod) / (U * U)
                    idx = x * NX * NY + y * NY + z
                    volume[idx] += val * weight

        # 5. Cleanup Image
        # clReleaseMemObject(image)

    # 6. Final Scaling
    scale = math.pi / P
    for i in range(len(volume)):
        volume[i] *= scale
        
    return volume

def hardware_interpolate(data, u, v, W, H):
    """
    Simulates OpenCL hardware bilinear interpolation.
    u and v are in continuous detector coordinates.
    """
    # Clamp to valid range
    if u < 0 or u >= W - 1 or v < 0 or v >= H - 1:
        return 0.0
        
    # Integer parts
    u0 = int(math.floor(u))
    v0 = int(math.floor(v))
    
    # Fractional parts
    fu = u - u0
    fv = v - v0
    
    # Neighbors
    # Layout is row-major: index = v * W + u
    val00 = data[v0 * W + u0]
    val01 = data[v0 * W + (u0 + 1)]
    val10 = data[(v0 + 1) * W + u0]
    val11 = data[(v0 + 1) * W + (u0 + 1)]
    
    # Bilinear interpolation
    top = val00 * (1 - fu) + val01 * fu
    bottom = val10 * (1 - fu) + val11 * fu
    return top * (1 - fv) + bottom * fv
```

### Edge Cases and Pitfalls
- **Boundary Conditions:** If $u_{\text{proj}}$ or $v_{\text{proj}}$ falls outside the detector bounds $[0, W]$ or $[0, H]$, the hardware interpolation typically returns 0.0. The pseudocode explicitly clamps and returns 0.0.
- **Division by Zero:** If $U=0$, the projection ray is parallel to the detector plane or originates from the source plane. The implementation must guard against this.
- **Memory Management:** The image object must be released after each projection to avoid memory leaks, as the loop creates a new image for each angle.

## Operating characteristics
- **Time Complexity:** $O(P \cdot NX \cdot NX \cdot NY)$. The dominant cost is the kernel execution for each projection.
- **Memory Complexity:** $O(W \cdot H + NX \cdot NX \cdot NY)$. The projection image requires $W \cdot H$ floats, and the volume requires $NX^2 NY$ floats.
- **Performance:** Typically faster than buffer-based backprojection due to the GPU's texture cache handling interpolation, reducing global memory bandwidth pressure.
- **Strengths:** Efficient interpolation; leverages hardware acceleration.
- **Limitations:** Requires pre-filtering on CPU (as per source notes); sequential projection processing limits parallelism across angles compared to fully parallel GPU approaches.

## Background
This method is part of a GPU-accelerated CT reconstruction toolkit. It specifically targets the backprojection stage using OpenCL image textures. The approach is documented in the `ct-reconstruction-gpu` repository.

## Sources
[S1] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
