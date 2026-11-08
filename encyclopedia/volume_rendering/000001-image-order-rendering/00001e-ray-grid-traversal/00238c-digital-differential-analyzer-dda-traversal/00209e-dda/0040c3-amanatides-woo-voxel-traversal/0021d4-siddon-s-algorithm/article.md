# Siddon's Algorithm

Also called: Exact discrete Beer-Lambert law, Ray casting with voxel intersection, Siddon ray tracing, exact radiological path calculation

## Overview
Siddon's Algorithm is a method for analytically computing the exact intersection lengths of a ray with each voxel in a 3D grid by tracking parametric alpha values at which the ray crosses parallel CT planes (x, y, z) [S1][S3, unofficial]. It serves as a ray-driven forward projection method that calculates the line-integral as a weighted sum of the values of intersected voxels, where weights are the length of intersection between each voxel and the ray [S2, unofficial]. This method provides an exact computation of the discretized Beer-Lambert law for X-ray attenuation, calculating the integral of linear attenuation coefficients (LACs) along a ray by summing the LAC in every voxel intersected by the ray, weighted by the intersection length [S3, unofficial]. It is widely used in slice-to-volume registration and differentiable rendering [S4][S3, unofficial].

## How it works
**Inputs and Outputs:**
The algorithm takes a ray defined by a source point $\mathbf{s}$ and a target/detector point $\mathbf{p}$, and a discrete 3D volume $\hat{\mu}$ (or $\mathbf{V}$) representing the voxel grid of Linear Attenuation Coefficients (LACs) or densities [S3, unofficial][S4].
Outputs typically include:
- `alphas`: Relative distance between start and endpoint for the intersections with the cube [S1].
- `l`: Lengths of intersections with cubes [S1].
- `rho`: Densities extracted from cubes [S1].
- `d12`: Distance between start and endpoint of ray tracing [S1].
- `ix`: Indices of hit voxels [S1].
- The final rendered intensity or energy attenuation $E(R)$ or $I_{\hat{\mu}}(\vec{r})$ [S3, unofficial][S4].

**Data Structures:**
The volume is represented as a 3D grid. The ray is parameterized by $\alpha \in [0, 1]$. The intersections are stored as a sorted set of parametric values $\{\alpha_1, \dots, \alpha_M\}$ [S3, unofficial][S4].

**Steps:**
1.  **Define the Ray:** Define the ray $\vec{r}(\alpha) = \mathbf{s} + \alpha(\mathbf{p} - \mathbf{s})$ for $\alpha \in [0, 1]$, where $\mathbf{s}$ is the X-ray source position and $\mathbf{p}$ is the detector pixel position [S3, unofficial][S4].
2.  **Compute Plane Intersections:** Calculate the intersection parameters of the ray with the parallel planes comprising the voxel grid. For a voxel size $\Delta X$ in the x-direction and the location of the 0-th plane $b_x$, the intersection with the $i$-th plane is:
    $$ \alpha_x(i) = \frac{b_x + i \Delta X - \mathbf{s}_x}{\mathbf{p}_x - \mathbf{s}_x} $$
    Analogous expressions are used for $\alpha_y(\cdot)$ and $\alpha_z(\cdot)$ [S4].
3.  **Sort Intersections:** Combine the intersection parameters from all three axes and sort them to form the set $\mathbf{\alpha} = \text{sort}(\mathbf{\alpha}_x, \mathbf{\alpha}_y, \mathbf{\alpha}_z)$, which contains $M$ values parameterizing the intersections [S4].
4.  **Iterate and Sum:** Iterate $m$ from 1 to $M-1$. For each segment between $\alpha_m$ and $\alpha_{m+1}$:
    a. Calculate the midpoint parameter: $\alpha_{mid} = \frac{\alpha_{m+1} + \alpha_m}{2}$ [S3, unofficial].
    b. Evaluate the voxel LAC at the midpoint: $\hat{\mu}[\mathbf{s} + \alpha_{mid} (\mathbf{p} - \mathbf{s})]$ [S3, unofficial]. Note: This indexing operation returns the LAC of the intersected voxel [S3, unofficial].
    c. Multiply by the segment length parameter difference: $(\alpha_{m+1} - \alpha_m)$ [S3, unofficial].
5.  **Final Calculation:** Sum these products and multiply by the ray length $\|\mathbf{p} - \mathbf{s}\|$ to get the rendered intensity [S3, unofficial]:
    $$ I_{\hat{\mu}}(\vec{r}) = \| \mathbf{p} - \mathbf{s} \| \sum_{m=1}^{M-1} \hat{\mu} \left[ \mathbf{s} + \frac{\alpha_{m+1} + \alpha_m}{2} (\mathbf{p} - \mathbf{s}) \right] (\alpha_{m+1} - \alpha_m) $$
    Alternatively expressed as energy attenuation $E(R)$ [S4]:
    $$ E(R) = \| \mathbf{p} - \mathbf{s} \|_2 \sum_{m=1}^{M-1} (\alpha_{m+1} - \alpha_m) \mathbf{V} \left[ \mathbf{s} + \frac{\alpha_{m+1} + \alpha_m}{2} (\mathbf{p} - \mathbf{s}) \right] $$
    The term $\|\mathbf{p} - \mathbf{s}\|_2$ cancels the reciprocal length units of $\mathbf{V}$ to make $E$ unitless [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def siddons_algorithm(source, target, volume, voxel_size, origin_offset):
    """
    source: tuple (sx, sy, sz)
    target: tuple (px, py, pz)
    volume: 3D array of LACs
    voxel_size: float, size of one voxel in mm
    origin_offset: tuple (bx, by, bz), location of the 0-th plane in mm
    """
    sx, sy, sz = source
    px, py, pz = target
    bx, by, bz = origin_offset
    dx, dy, dz = voxel_size

    # 1. Define Ray Direction and Length
    dir_x = px - sx
    dir_y = py - sy
    dir_z = pz - sz
    ray_length = (dir_x**2 + dir_y**2 + dir_z**2)**0.5

    # 2. Compute Intersection Parameters for each axis
    # Avoid division by zero if ray is parallel to an axis
    eps = 1e-08
    
    alphas_x = []
    if abs(dir_x) > eps:
        # Determine min and max i for the volume bounds
        # Assuming volume spans from origin_offset to origin_offset + N*voxel_size
        # For simplicity, we compute intersections with all planes within the ray's bounding box
        # In practice, one would compute the range of i such that alpha is in [0, 1]
        # Here we generate a sufficient range and filter later
        min_i = int(((bx - sx) / dir_x) - 1) if dir_x > 0 else int(((bx - sx) / dir_x) + 1)
        max_i = int(((bx + len(volume)*dx - sx) / dir_x) + 1) if dir_x > 0 else int(((bx + len(volume)*dx - sx) / dir_x) - 1)
        
        for i in range(min_i, max_i + 1):
            alpha = (bx + i * dx - sx) / dir_x
            if 0.0 <= alpha <= 1.0:
                alphas_x.append(alpha)
    else:
        # Ray is parallel to x-planes; no intersections or infinite. 
        # If inside the slab, it intersects all planes? 
        # Standard Siddon implementation usually handles this by skipping or treating as no intersection 
        # if the ray doesn't cross the plane boundaries.
        pass

    alphas_y = []
    if abs(dir_y) > eps:
        min_j = int(((by - sy) / dir_y) - 1) if dir_y > 0 else int(((by - sy) / dir_y) + 1)
        max_j = int(((by + len(volume[0])*dy - sy) / dir_y) + 1) if dir_y > 0 else int(((by + len(volume[0])*dy - sy) / dir_y) - 1)
        for j in range(min_j, max_j + 1):
            alpha = (by + j * dy - sy) / dir_y
            if 0.0 <= alpha <= 1.0:
                alphas_y.append(alpha)

    alphas_z = []
    if abs(dir_z) > eps:
        min_k = int(((bz - sz) / dir_z) - 1) if dir_z > 0 else int(((bz - sz) / dir_z) + 1)
        max_k = int(((bz + len(volume[0][0])*dz - sz) / dir_z) + 1) if dir_z > 0 else int(((bz + len(volume[0][0])*dz - sz) / dir_z) - 1)
        for k in range(min_k, max_k + 1):
            alpha = (bz + k * dz - sz) / dir_z
            if 0.0 <= alpha <= 1.0:
                alphas_z.append(alpha)

    # 3. Sort all intersections
    all_alphas = sorted(alphas_x + alphas_y + alphas_z)
    
    # Ensure start and end are included if not already
    if not all_alphas or all_alphas[0] > 0.0:
        all_alphas.insert(0, 0.0)
    if not all_alphas or all_alphas[-1] < 1.0:
        all_alphas.append(1.0)
        
    # Remove duplicates and sort again
    all_alphas = sorted(list(set(all_alphas)))

    # 4. Iterate and Sum
    total_integral = 0.0
    for m in range(len(all_alphas) - 1):
        alpha_m = all_alphas[m]
        alpha_m1 = all_alphas[m+1]
        
        # Midpoint parameter
        alpha_mid = (alpha_m1 + alpha_m) / 2.0
        
        # Midpoint position in world coordinates
        mid_x = sx + alpha_mid * dir_x
        mid_y = sy + alpha_mid * dir_y
        mid_z = sz + alpha_mid * dir_z
        
        # Map to voxel indices
        # Assuming voxel_shift=0.5 means center of voxel
        # Index = (coord - origin) / voxel_size - 0.5
        ix = int((mid_x - bx) / dx - 0.5)
        iy = int((mid_y - by) / dy - 0.5)
        iz = int((mid_z - bz) / dz - 0.5)
        
        # Check bounds
        if 0 <= ix < len(volume) and 0 <= iy < len(volume[0]) and 0 <= iz < len(volume[0][0]):
            mu = volume[ix][iy][iz]
            segment_length_param = alpha_m1 - alpha_m
            total_integral += mu * segment_length_param
        else:
            # [G] If the midpoint falls outside the volume bounds, the contribution is 0.
            pass

    # 5. Final Intensity
    intensity = ray_length * total_integral
    return intensity
```

**Parameters:**
- `voxel_shift`: float, default 0.5. Determines if the voxel is at the top left corner (0) or the center (0.5) [S4].
- `mode`: str, default 'nearest'. Interpolation mode for grid_sample [S4].
- `filter_intersections_outside_volume`: bool, default False. If True, uses alphamin/max to filter the intersections [S4].
- `eps`: float, default 1e-08. Small constant to avoid div by zero errors [S4].
- `isocenterCube`: The isocenter position in cube coordinates [mm] [S1].
- `resolution`: The resolution of the cubes in mm/voxel [S1].
- `sourcePoint`: The starting point of the ray tracing [S1].
- `targetPoint`: The ending point of the ray tracing [S1].
- `cubes`: A cell array of density/material cubes to trace through [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Division by zero must be handled if the ray is parallel to a grid plane [S4].
- **Boundaries:** Intersections outside the volume should be filtered if `filter_intersections_outside_volume` is True [S4].
- **Degenerate Input:** If the ray does not intersect the volume, the sum is zero.
- **Accuracy:** Nearest-neighbour interpolation is too simplified for most applications; the input voxel volume must be at least double the resolution in each dimension compared to the desired reconstruction resolution to maintain accuracy [S2, unofficial].
- **Overfitting:** In sparse-view settings, Siddon's method suffers from overfitting because it does not allow ground truth pixels to provide self-supervision to voxels not directly intersected by the ray [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Computation scales cubically with the resolution of the voxelgrid $\hat{\mu}$ because it requires computing every plane intersection [S3, unofficial]. The number of intersections $M$ is proportional to the number of voxels intersected, which can be up to $N_x + N_y + N_z$ for a grid of size $N_x \times N_y \times N_z$.

**Measured Performance:**
On 40 test walnuts with 15 input views, Siddon's method achieved SSIM 0.923, PSNR 37.69, MSE $1.28 \times 10^{-5}$, PCC 0.96, and runtime 11.3 minutes on a single NVIDIA RTX A6000 GPU [S3, unofficial]. The batch size is limited by memory constraints to $|\mathcal{R}| = 5.5 \times 10^5$ rays per batch [S3, unofficial].

**Strengths and Limitations:**
- **Strengths:** Exactly computes line integrals, leading to higher fidelity novel views compared to trilinear interpolation [S3, unofficial]. It is widely used in slice-to-volume registration [S4].
- **Limitations:** Inferior in accuracy to Joseph's and Köhler's methods when compared against ideal analytical values [S2, unofficial]. It does not account for patterns of reflection and scattering present in real X-ray systems [S4]. It is more prone to overfitting in sparse views compared to trilinear interpolation [S3, unofficial].

**When to Use:**
Use when exact radiological path calculation is required for differentiable rendering or slice-to-volume registration, and computational resources allow for the cubic scaling [S3, unofficial][S4].

**When Not to Use:**
Avoid when real-time performance is critical and approximate methods (like trilinear interpolation) are sufficient, or when sparse-view reconstruction is prone to overfitting [S3, unofficial].

## Background
Siddon's Algorithm was introduced by R.L. Siddon in 1985 in the paper 'Fast calculation of the exact radiological path for a three-dimensional CT array', published in Med. Phys. 12(2):252-5 [S1][S3, unofficial]. It is a parametric method to generate Digital Radiographic Reconstructions (DRRs) by modeling the geometry of an idealized projectional radiography system [S4]. It is used in medical imaging, particularly in radiation therapy planning (e.g., matRad) and CBCT reconstruction [S1][S3, unofficial].

## Sources
[S1] rayTracing — matRad 3.2.3 documentation, https://matrad.readthedocs.io/en/stable/api/raytracing.html, fetched 2026-10-25
[S2] 5 Forward Projection through Voxel Volumes, https://www.cvl.isy.liu.se/education/undergraduate/tsbb31/download/TurbellPhDCh5.pdf, fetched 2026-10-25 (unofficial)
[S3] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S4] renderers – diffdrr, https://vivekg.dev/DiffDRR/api/renderers.html, fetched 2026-10-29
