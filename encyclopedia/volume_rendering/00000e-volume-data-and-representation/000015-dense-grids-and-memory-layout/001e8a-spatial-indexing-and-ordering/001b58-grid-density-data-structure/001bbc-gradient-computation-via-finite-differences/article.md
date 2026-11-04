# Gradient Computation via Finite Differences

Also called: Central Difference Gradient, Central Difference Gradient Estimation, Central-Difference Gradient Operator, Component-wise Linear Gradient Estimation, Finite Difference Gradient, Gradient Computation Using Central Differences, Numerical Gradient Approximation

## Overview
Gradient Computation via Finite Differences estimates the spatial gradient vector of a scalar volume field at sample positions to enable local illumination (e.g., Phong shading) and multidimensional transfer functions [S1, unofficial][S3, book]. It sets itself apart from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by requiring no basis function derivatives and from [Sobel Operator](1416-sobel-operator.md) by using a minimal 6-neighbor stencil rather than a separable smoothing kernel, making it faster but less robust to noise [S2, S7, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar volume grid $V$ of dimensions $N_x \times N_y \times N_z$, stored in row-major order (x fastest, y middle, z slowest) [S6, unofficial].
- Output: A gradient vector grid $G$ of the same dimensions, where each voxel stores a 3D vector $(g_x, g_y, g_z)$ [S5, unofficial].

**Data Structures:**
- The gradient array is a 1D float array of size $N_x N_y N_z \times 3$, with interleaved RGB components per voxel: `gradient[idx*3]` is $g_x$, `gradient[idx*3+1]` is $g_y$, `gradient[idx*3+2]` is $g_z$ [S5, unofficial].
- Alternatively, for GPU storage, gradients are encoded into signed normalized 8-bit textures (e.g., `GL_RGBA8_SNORM`) where the direction is mapped to $[0,1]$ and magnitude may be stored in an alpha channel or separate buffer [S4, unofficial].

**Parameters:**
- $h$: The physical distance between voxels along the axis of differentiation (voxel size) [S3, book][S16, unofficial].
- `grad_magnitude_modifier`: A scalar multiplier applied to the gradient magnitude before clamping (typical value depends on data range, often 1.0 or scaled to fit texture range) [S4, unofficial].

**Algorithm Steps:**
1. Iterate over every voxel $(x, y, z)$ in the volume grid [S5, S8, unofficial].
2. For each axis, compute the central difference approximation of the partial derivative [S1, unofficial].
   - **X-component:** Sample the voxel at $(x-1, y, z)$ and $(x+1, y, z)$ [S2, unofficial].
     $$ g_x = \frac{V(x+1, y, z) - V(x-1, y, z)}{2h} $$ [S3, book][S17]
   - **Y-component:** Sample the voxel at $(x, y-1, z)$ and $(x, y+1, z)$ [S2, unofficial].
     $$ g_y = \frac{V(x, y+1, z) - V(x, y-1, z)}{2h} $$ [S3, book][S17]
   - **Z-component:** Sample the voxel at $(x, y, z-1)$ and $(x, y, z+1)$ [S2, unofficial].
     $$ g_z = \frac{V(x, y, z+1) - V(x, y, z-1)}{2h} $$ [S3, book][S17]
3. If the gradient is used solely for surface normals, the division by $2h$ may be omitted, as the vector is normalized immediately after [S10, unofficial].
4. Handle boundary conditions: At data boundaries, the gradient is undefined because neighbors are missing. Common handling includes padding the volume by repeating boundary values [S3, book][S16, unofficial].
5. Normalize the resulting vector $\mathbf{g} = (g_x, g_y, g_z)$ to obtain the surface normal $\mathbf{n}$:
   $$ \mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|} \quad \text{if } \|\mathbf{g}\| > \epsilon $$
   If $\|\mathbf{g}\| \le \epsilon$ (e.g., $10^{-6}$), set $\mathbf{n} = (0, 0, 0)$ or a default normal [S10, unofficial].
6. Store $\mathbf{n}$ (and optionally magnitude) in the output gradient grid or texture [S5, S10, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_central_difference_gradients(volume, resolution, voxel_size):
    """
    Computes the gradient vector for each voxel in a scalar volume using central differences.
    
    Args:
        volume: 3D list of shape (res_x, res_y, res_z) containing scalar values.
        resolution: Tuple (res_x, res_y, res_z).
        voxel_size: Float representing the physical distance between voxels (h).
        
    Returns:
        gradients: 3D list of shape (res_x, res_y, res_z, 3) containing normalized gradient vectors.
    """
    res_x, res_y, res_z = resolution
    gradients = [[[ [0.0, 0.0, 0.0] for _ in range(res_z)] for _ in range(res_y)] for _ in range(res_x)]
    inv_2h = 1.0 / (2.0 * voxel_size)
    
    for z in range(res_z):
        for y in range(res_y):
            for x in range(res_x):
                # Pad boundaries by repeating edge values [S3, book]
                x_prev = x - 1 if x > 0 else 0
                x_next = x + 1 if x < res_x - 1 else res_x - 1
                y_prev = y - 1 if y > 0 else 0
                y_next = y + 1 if y < res_y - 1 else res_y - 1
                z_prev = z - 1 if z > 0 else 0
                z_next = z + 1 if z < res_z - 1 else res_z - 1
                
                gx = (volume[x_next][y][z] - volume[x_prev][y][z]) * inv_2h
                gy = (volume[x][y_next][z] - volume[x][y_prev][z]) * inv_2h
                gz = (volume[x][y][z_next] - volume[x][y][z_prev]) * inv_2h
                
                length = (gx*gx + gy*gy + gz*gz) ** 0.5
                if length > 1e-6:
                    gradients[x][y][z][0] = gx / length
                    gradients[x][y][z][1] = gy / length
                    gradients[x][y][z][2] = gz / length
                else:
                    gradients[x][y][z][0] = 0.0
                    gradients[x][y][z][1] = 0.0
                    gradients[x][y][z][2] = 0.0
    return gradients
```

**Edge Cases and Pitfalls:**
- **Boundary Access:** The standard central difference formula requires neighbors at $i-1$ and $i+1$. Accessing these at the grid edges causes out-of-bounds memory errors unless clamping, padding, or reflection is applied [S6, S13, unofficial].
- **Zero Magnitude:** In homogeneous regions, the gradient magnitude is zero. Normalization must check for this to avoid division by zero [S10, unofficial].
- **Noise Sensitivity:** Because it uses only 6 neighbors, the central difference operator produces less smooth gradients compared to operators using more voxels (like Sobel), making it sensitive to high-frequency noise [S2, S7, unofficial].
- **Visual Artifacts:** Artifacts similar to those from trilinear interpolation resampling may occur [S16, unofficial].

## Subtopics

### [Gradient Map Generation](1204-gradient-map-generation.md)
A compute shader pass that calculates gradients for all voxels and stores them in a gradient map image, including magnitude and direction. It uses a 3D image buffer with `rgba8_snorm` layout, storing normalized gradient direction in x, y, z and magnitude in w. The magnitude is clamped to [0, 1] after applying a modifier.

### Voxel Gradient Calculation
A pre-processing step using the Sobel operator to detect edges within the volume and compute gradients. Gradient magnitude modulates voxel opacity to hide homogeneous regions, and gradient direction is used for shading [S19, unofficial].

### Clamped Central Differences
A variant of central differencing for gradient computation that clamps offset positions to the current region to avoid additional ray traversals. It computes six additional basis reconstruction samples at positions offset from the current sample by a distance proportional to the region's sample rate. Gradients are not necessarily continuous at level boundaries because the offset size can change. Accurate central differencing requires tracing 6 additional rays per sample via infinitesimal rays to find containing regions, which is expensive. The clamped version removes ray traversals but reduces quality [S20].

### Precomputed Gradient Volume
A method where gradients are precomputed and stored alongside scalar values in a flat array. For each voxel at $(x, y, z)$, neighbors are fetched using a safe access function (returning 0.0 for out-of-bounds indices, effectively zero-padding boundaries). The gradient vector is normalized and stored in a union with the scalar value. Template parameter defines the cubic volume dimension, with a typical code value of 60 (`CUBE_SIZE`). `VoxelVertex` is a union containing `vec4 xyzw` and `vec3 gradient`, where `xyzw.w` stores the scalar volume value and `gradient` stores the precomputed normalized gradient vector. Data is stored in a flat `std::array<VoxelVertex, cube(length)> m_data` indexed by `cartesian_to_index(v, length) = v.z * length * length + v.y * length + v.x` [S21, unofficial].

### Gradient-based Surface Projection
Visualization of interfaces between two regions of different density by projecting through a matrix of gradient magnitude values derived from voxel density. The estimate of an interface between regions at each voxel is defined by a surface normal $N$ and surface strength $|N|$, derived from voxel density. A refractive index-only image is obtained by projecting directly through the surface magnitude information. Surface normal $N = (N_x, N_y, N_z)$ is approximated by a finite difference of neighbouring densities, and surface strength $|N|$ is the L2-norm of $N$. Surfaces may appear to lie at sub-voxel accuracy because they are defined by fractional values at each voxel, allowing visualization of diffuse transitions not possible with binary representation [S22, unofficial].

### Curl Grid Computation
Computes the curl $\nabla \times \mathbf{F}$ at every active voxel of a vector grid using central differences. It initializes a `GradStencil` with the grid tree, iterates over all active voxels, moves the stencil to the current coordinate, and computes the x, y, and z components of the curl using the formulas. The result is stored in a dictionary keyed by coordinate and built into a result grid with a zero vector background and the original grid's voxel size. The formulas are: `curl_x = (v[4][3] - v[5][3] - v[6][2] + v[7][2]) * 0.5`, `curl_y = (v[6][1] - v[7][1] - v[2][3] + v[3][3]) * 0.5`, `curl_z = (v[2][2] - v[3][2] - v[4][1] + v[5][1]) * 0.5`. The scalar multiplier `0.5` represents $1/(2 \times \text{voxel\_size})$ in index space [S9, unofficial].

### Laplacian Grid Computation
Computes the Laplacian $\nabla^2 f$ at every active voxel using a 6-point stencil in index space. It initializes a `GradStencil` with the grid tree, iterates over all active voxels, moves the stencil to the current coordinate, computes the Laplacian scalar using the stencil, stores it in a dictionary keyed by coordinate, and builds the result grid with a zero background and the original grid's voxel size [S9, unofficial].

### Gradient Magnitude Computation
Computes the Euclidean norm $|\mathbf{v}|$ at every active voxel of a vector grid. It iterates over all active voxels, computes the norm of the vector, stores it in a dictionary keyed by coordinate, and builds the result grid with a zero background and the original grid's voxel size [S9, unofficial].

### Forward Difference Gradient
Gradient estimation taking four neighboring voxels as input. For a voxel $V$ at $(x, y, z)$, the gradient components are calculated using forward differences: $\nabla_x = V_{x+1,y,z} - V_{x,y,z}$, $\nabla_y = V_{x,y+1,z} - V_{x,y,z}$, $\nabla_z = V_{x,y,z+1} - V_{x,y,z}$ [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as each voxel is processed once with a constant number of operations [S5, unofficial].
- **Memory Complexity:** $O(N)$ for storing the gradient grid, typically 3 times the size of the scalar volume if stored as floats [S5, unofficial].
- **Performance:** Requires 6 sampling points per voxel. It is faster than Sobel3D (which requires 27 samples) but provides lower quality gradient descriptions [S2, unofficial].
- **Strengths:** Simple to implement, low computational cost, suitable for real-time applications where noise is low or pre-smoothing is applied [G].
- **Limitations:** Sensitive to noise due to small stencil size; boundary handling requires care; less smooth than larger kernels [S2, S7, unofficial].

## Background
In volume rendering, the method was popularized for adding visual cues like directional light and Phong shading to ray-traced volumes [S1, unofficial]. It is used in scientific visualization tools and libraries, such as CGAL's isosurfacing modules [S18].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S3] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S4] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
[S5] extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, fetched 2026-10-26 (unofficial)
[S6] thoniorf/volumeraycasting, https://github.com/thoniorf/volumeraycasting, fetched 2026-10-26 (unofficial)
[S7] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S8] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S9] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S10] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S11] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S12] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S13] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
[S14] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S15] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S16] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29 (unofficial)
[S17] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S18] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26
[S19] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30 (unofficial)
[S20] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26
[S21] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
