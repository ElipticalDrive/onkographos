# Tri-linear Interpolation via LRP

Also called: Tri-linear Interpolation, LRP-based Interpolation

## Overview
Tri-linear interpolation via LRP is a GPU-accelerated method for sampling volumetric data stored in 3D textures, estimating the scalar value at arbitrary floating-point coordinates by linearly interpolating the eight surrounding voxel values [S2, S3, unofficial]. It solves the problem of resampling discrete voxel grids into continuous space for volume rendering, providing smoother structural details than direct voxel rendering [S2, S3, unofficial]. It sets itself apart from higher-order methods by using only linear blending factors and from CPU-based raycasting by leveraging hardware rasterization to interpolate texture coordinates "for free" between vertices [S3, unofficial].

## How it works
**Inputs:**
- A 3D texture containing volumetric data with dimensions $W \times H \times D$ [S3, unofficial].
- A 3D texture coordinate $(u, v, w)$ for the resampling point, where $u, v, w \in [0, 1]$ [S3, unofficial].

**Outputs:**
- The interpolated scalar value $S_X$ at the query point [S2, unofficial].

**Data Structures:**
- **3D Texture:** Stores the volume data. Accessed via hardware texture units [S3, unofficial].
- **Vertex Attributes:** Eight texture-coordinate attributes per vertex, storing the pre-computed texture coordinates for the eight nearest neighbors of the vertex [S3, unofficial].
- **Interpolated Coordinates:** During rasterization, the GPU interpolates the eight vertex attributes across the primitive, providing eight distinct texture coordinates $(u_i, v_i, w_i)$ for each fragment [S3, unofficial].

**Steps:**
1. **Pre-computation (Per-Vertex Stage):** For each vertex of the proxy geometry, calculate the texture coordinates of its eight nearest neighbors in the 3D texture grid. Store these in eight separate texture-coordinate vertex attributes [S3, unofficial].
2. **Rasterization:** The GPU interpolates these eight sets of texture coordinates across the rendered primitive. Each fragment receives eight interpolated texture coordinates corresponding to the eight corners of the voxel cell containing the fragment's position [S3, unofficial].
3. **Blending Factor Calculation (Per-Fragment Stage):** For each of the eight interpolated texture coordinates $(u, v, w)$, calculate the blending factors. Multiply the texture coordinate by the original texture resolution $(W, H, D)$ to get absolute texel coordinates. Take the fractional part of these coordinates to determine the interpolation weights [S3, unofficial].
   $$ \alpha = \text{frac}(u \cdot W) $$
   $$ \beta = \text{frac}(v \cdot H) $$
   $$ \gamma = \text{frac}(w \cdot D) $$
   Where $\text{frac}(x) = x - \lfloor x \rfloor$ [S2, S3, unofficial].
4. **Voxel Fetching:** Fetch the scalar values $S_{ijk}$ from the 3D texture using the eight interpolated texture coordinates. The indices correspond to the integer parts of the absolute coordinates [S2, S3, unofficial].
   $$ S_{000} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor, \lfloor vH \rfloor, \lfloor wD \rfloor) $$
   $$ S_{100} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor + 1, \lfloor vH \rfloor, \lfloor wD \rfloor) $$
   $$ S_{010} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor, \lfloor vH \rfloor + 1, \lfloor wD \rfloor) $$
   $$ S_{110} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor + 1, \lfloor vH \rfloor + 1, \lfloor wD \rfloor) $$
   $$ S_{001} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor, \lfloor vH \rfloor, \lfloor wD \rfloor + 1) $$
   $$ S_{101} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor + 1, \lfloor vH \rfloor, \lfloor wD \rfloor + 1) $$
   $$ S_{011} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor, \lfloor vH \rfloor + 1, \lfloor wD \rfloor + 1) $$
   $$ S_{111} = \text{tex3D}(u, v, w) \text{ at } (\lfloor uW \rfloor + 1, \lfloor vH \rfloor + 1, \lfloor wD \rfloor + 1) $$
5. **Interpolation:** Compute the final value $S_X$ using the trilinear formula [S2, unofficial]:
   $$ S_X = (1-\alpha)(1-\beta)(1-\gamma)S_{000} + \alpha(1-\beta)(1-\gamma)S_{100} + (1-\alpha)\beta(1-\gamma)S_{010} + \alpha\beta(1-\gamma)S_{110} + (1-\alpha)(1-\beta)\gamma S_{001} + \alpha(1-\beta)\gamma S_{101} + (1-\alpha)\beta\gamma S_{011} + \alpha\beta\gamma S_{111} $$

**Parameters:**
- $u, v, w$: 3D texture coordinates of the resampling point, range $[0, 1]$ [S3, unofficial].
- $W, H, D$: Original dimensions (width, height, depth) of the 3D texture [S3, unofficial].

**Pitfalls:**
- **Memory Bandwidth:** Trilinear interpolation requires eight memory reads per sample point. For real-time rendering, this creates a high demand for memory bandwidth, potentially slowing down rendering speed if memory latency is high [S1, unofficial].
- **Boundary Handling:** If the ceiling coordinate exceeds the volume dimension (e.g., $\lfloor uW \rfloor + 1 \ge W$), it must be clamped to the maximum index ($W-1$) to prevent out-of-bounds array access [S2, unofficial].
- **Precision:** The LRP instruction relies on accurate interpolation of texture coordinates. Precision errors in coordinate interpolation can lead to incorrect blending factors [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tri_linear_interpolation_via_lrp(tex_coord, resolution, texture_fetch):
    """
    Performs tri-linear interpolation using the LRP method.
    
    Args:
        tex_coord (tuple): (u, v, w) texture coordinates in [0, 1].
        resolution (tuple): (W, H, D) dimensions of the 3D texture.
        texture_fetch (function): Function to fetch voxel value given integer indices (x, y, z).
        
    Returns:
        float: Interpolated scalar value.
    """
    u, v, w = tex_coord
    W, H, D = resolution
    
    # Calculate absolute coordinates
    abs_u = u * W
    abs_v = v * H
    abs_w = w * D
    
    # Calculate integer floor and ceiling coordinates
    xf = int(abs_u)
    xc = min(xf + 1, W - 1)  # Clamp to max index
    yf = int(abs_v)
    yc = min(yf + 1, H - 1)  # Clamp to max index
    zf = int(abs_w)
    zc = min(zf + 1, D - 1)  # Clamp to max index
    
    # Calculate blending factors (fractional parts)
    alpha = abs_u - xf
    beta = abs_v - yf
    gamma = abs_w - zf
    
    # Fetch the eight corner voxel values
    S000 = texture_fetch(xf, yf, zf)
    S100 = texture_fetch(xc, yf, zf)
    S010 = texture_fetch(xf, yc, zf)
    S110 = texture_fetch(xc, yc, zf)
    S001 = texture_fetch(xf, yf, zc)
    S101 = texture_fetch(xc, yf, zc)
    S011 = texture_fetch(xf, yc, zc)
    S111 = texture_fetch(xc, yc, zc)
    
    # Compute interpolated value
    S_X = (1-alpha)*(1-beta)*(1-gamma)*S000 + \
          alpha*(1-beta)*(1-gamma)*S100 + \
          (1-alpha)*beta*(1-gamma)*S010 + \
          alpha*beta*(1-gamma)*S110 + \
          (1-alpha)*(1-beta)*gamma*S001 + \
          alpha*(1-beta)*gamma*S101 + \
          (1-alpha)*beta*gamma*S011 + \
          alpha*beta*gamma*S111
          
    return S_X
```

## Subtopics

### [Coarsest-Level Lerp](2770-coarsest-level-lerp.md)
Coarsest-Level Lerp is a method for computing the value of a coarser-side octant vertex in Adaptive Mesh Refinement (AMR) volume rendering by performing trilinear interpolation on the coarsest refinement level involved in the input data. It solves the problem of defining continuous boundary values for octants that abut coarser regions by ensuring the interpolant is based on the lowest-resolution grid that covers the spatial extent of the query. It sets itself apart from Finest Level Lerp by prioritizing the coarsest available data rather than the finest, and from Current-Level Lerp by using a fixed global level rather than an adaptive leaf-level lookup.

### Index of Refraction Update via Trilinear Interpolation
A method to update a 3D volume texture of index of refraction (IOR) values based on particle positions, using trilinear interpolation to distribute a constant IOR increment (0.1) to the 8 surrounding grid cells [S4, unofficial]. It involves resetting the IOR volume, calculating floating-point cell coordinates for each particle, and updating the 8 neighboring cells using trilinear weights based on the fractional part of the cell coordinates [S4, unofficial].

### Tensor Core Accelerated Texture Interpolation
An attempt to use GPU matrix-multiplication units (Tensor Cores) to perform texture interpolation, outside of artificial neural networks [S5, unofficial]. This approach was found to be not very performant compared to standard methods, as matrix-multiplication units only work with predefined matrix sizes, achieve peak efficiency only with heavily reduced precision number formats, and are complicated to feed, making them difficult to use for applications that do not fit within those narrow constraints [S5, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample point, as it involves a fixed number of arithmetic operations and memory fetches [S2, S3, unofficial].
- **Memory Complexity:** $O(1)$ per sample point, requiring storage for 8 voxel values and blending factors [S1, unofficial].
- **Measured Performance:** Requires at least eight memory reads per sample point. For real-time rendering, this can result in millions or billions of memory reads per frame, creating a significant bottleneck if memory bandwidth is insufficient [S1, unofficial].
- **Scaling:** Performance scales linearly with the number of sample points, but is heavily dependent on memory bandwidth and latency [S1, unofficial].
- **Strengths:** Provides smooth rendering of volumetric data, avoiding the blocky appearance of direct voxel rendering [S2, S3, unofficial]. Hardware-accelerated via LRP instruction and texture units [S3, unofficial].
- **Limitations:** High memory bandwidth requirement. Less structural detail compared to higher-order interpolation methods [S2, S3, unofficial].
- **When to use:** Real-time volume rendering where smoothness is prioritized over extreme detail, and hardware supports efficient 3D texture sampling [S1, S3, unofficial].
- **When not to use:** When memory bandwidth is severely constrained and higher-order interpolation is required for visual fidelity [S2, S3, unofficial].

## Background
Tri-linear interpolation is the primary interpolation method for volume rendering, providing smooth rendering of volumetric data [S1, unofficial]. It is a method of multivariate interpolation on a 3-D regular grid that approximates the value of a function at an intermediate point linearly, using function data on the lattice points [S2, unofficial]. The GPU-based implementation using the LRP instruction was described by Viola et al. [2004] in the context of hardware-accelerated frequency domain volume rendering [S3, unofficial]. It has been widely adopted in real-time volume rendering systems due to its balance of quality and performance [S1, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S3] PDF Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S4] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S5] Real-Time Rendering of Ultra-Large and Ultra-Detailed Virtual World using Voxels, https://hal.science/tel-05803730v1, fetched 2026-10-26 (unofficial)
