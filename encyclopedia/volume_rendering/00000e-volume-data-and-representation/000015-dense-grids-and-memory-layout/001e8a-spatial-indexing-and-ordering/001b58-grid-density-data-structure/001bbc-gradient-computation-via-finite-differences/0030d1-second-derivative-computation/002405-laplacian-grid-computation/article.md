# Laplacian Grid Computation
Also called: Laplacian of Gaussian (LoG) Scale Detection, Normalized Laplacian Detection

## Overview
Computes the Laplacian scalar field $\nabla^2 f$ for every active voxel in a volume using a 6-point finite difference stencil [S1, unofficial]. It serves as the foundation for scale-space blob detection, where local maxima of the normalized Laplacian identify feature sizes [S2, unofficial]. It differs from sibling methods like the [Sobel Operator](1416-sobel-operator.md) by computing the second derivative (curvature) rather than the first derivative (gradient), and from [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) by using the trace of the Hessian rather than the mean curvature tensor.

## How it works
**Inputs:**
- Scalar volume grid $V$ with dimensions $W \times H \times D$.
- Grid tree or active voxel list defining the domain of interest.
- Voxel size $\Delta x$ (assumed isotropic for the standard 6-point stencil).

**Outputs:**
- Laplacian grid $L$ of same dimensions as $V$, storing $\nabla^2 V$ at each voxel.

**Data Structures:**
- `GradStencil`: A helper structure holding the grid tree and coordinate iterator.
- `LaplacianMap`: A dictionary or sparse array keyed by voxel coordinate $(i, j, k)$ mapping to a `Float64` value.

**Steps:**
1. Initialize a `GradStencil` with the input grid tree.
2. Iterate over all active voxels $(i, j, k)$ in the grid tree.
3. For each voxel, compute the Laplacian using the 6-point stencil.
4. Store the result in `LaplacianMap` keyed by $(i, j, k)$.
5. Build the result grid $L$ from `LaplacianMap`, using a background value of zero and the original grid's voxel size.

**Formula:**
The Laplacian is the trace of the Hessian matrix $H(x)$ [S2, unofficial]:
$$
\nabla^2 f = \text{Tr}(H(x)) = \frac{\partial^2 f}{\partial x^2} + \frac{\partial^2 f}{\partial y^2} + \frac{\partial^2 f}{\partial z^2}
$$
Using central differences with voxel size $\Delta x$:
$$
\nabla^2 f(i,j,k) \approx \frac{f(i+1,j,k) + f(i-1,j,k) - 2f(i,j,k)}{\Delta x^2} + \frac{f(i,j+1,k) + f(i,j-1,k) - 2f(i,j,k)}{\Delta x^2} + \frac{f(i,j,k+1) + f(i,j,k-1) - 2f(i,j,k)}{\Delta x^2}
$$
For scale detection, the **normalized Laplacian** is used [S2, unofficial]:
$$
t \nabla^2 L = t (L_{xx} + L_{yy} + L_{zz}) = t \text{Tr}(H(x))
$$
where $t$ is the scale parameter (variance of the Gaussian kernel). In the basic grid computation, $t=1$ (or $\Delta x^2$ depending on normalization convention) is often applied to stabilize the magnitude across scales.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_laplacian_grid(volume: VolumeGrid) -> LaplacianGrid:
    """
    Computes the Laplacian grid using a 6-point stencil.
    volume: VolumeGrid with fields .data (3D array), .voxel_size (float), .active_voxels (list of tuples)
    Returns: LaplacianGrid with fields .data (3D array), .voxel_size (float)
    """
    W, H, D = volume.data.shape
    dx = volume.voxel_size
    inv_dx2 = 1.0 / (dx * dx)
    
    # Initialize output grid with background zero
    laplacian_data = zeros_like(volume.data)
    
    # Iterate over active voxels
    for (i, j, k) in volume.active_voxels:
        # Compute second derivatives using central differences
        # Check boundaries: if neighbor is out of bounds, treat value as 0 or clamp?
        # Standard implementation assumes valid neighbors or handles boundaries explicitly.
        # Here we assume interior points or that volume handles boundary conditions.
        
        # x-direction
        f_ip = volume.data[i+1, j, k] if i+1 < W else 0.0
        f_im = volume.data[i-1, j, k] if i-1 >= 0 else 0.0
        d2f_dx2 = (f_ip + f_im - 2.0 * volume.data[i, j, k]) * inv_dx2
        
        # y-direction
        f_jp = volume.data[i, j+1, k] if j+1 < H else 0.0
        f_jm = volume.data[i, j-1, k] if j-1 >= 0 else 0.0
        d2f_dy2 = (f_jp + f_jm - 2.0 * volume.data[i, j, k]) * inv_dx2
        
        # z-direction
        f_kp = volume.data[i, j, k+1] if k+1 < D else 0.0
        f_km = volume.data[i, j, k-1] if k-1 >= 0 else 0.0
        d2f_dz2 = (f_kp + f_km - 2.0 * volume.data[i, j, k]) * inv_dx2
        
        # Sum to get Laplacian
        laplacian_data[i, j, k] = d2f_dx2 + d2f_dy2 + d2f_dz2
        
    return LaplacianGrid(data=laplacian_data, voxel_size=dx)
```

**Parameters:**
- `voxel_size`: Must be positive. If not isotropic, separate $\Delta x, \Delta y, \Delta z$ must be used in the denominator.
- `t` (scale): For normalized detection, $t$ is the scale parameter. Typical values depend on the feature size relative to voxel size.

**Edge Cases:**
- **Boundaries:** The 6-point stencil requires neighbors. At grid boundaries, values are typically treated as zero (Dirichlet) or the stencil is adjusted. The pseudocode above assumes zero-padding [G].
- **Degenerate Input:** If the volume is constant, the Laplacian is zero everywhere.
- **Precision:** Floating-point errors can accumulate in large volumes. Use `Float64` for intermediate calculations.

## Subtopics

### [GPU Laplacian Scale-Space Maxima Detection](5852-gpu-laplacian-scale-space-maxima-detection.md)
A GPU-based algorithm to compute a scale field by detecting local maxima of the Laplacian across a scale space, encoding derivative information in RGBA channels to avoid multiple passes for neighbor comparison.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of active voxels, as each voxel is processed once with a constant number of operations (6 reads, 12 adds, 3 muls).
- **Memory Complexity:** $O(N)$ for the output grid.
- **Performance:** Fast and parallelizable. Suitable for real-time or near-real-time volume exploration if the grid is sparse.
- **Strengths:** Simple to implement, robust for blob detection, provides scale information.
- **Limitations:** Sensitive to noise (second derivatives amplify high-frequency noise), requires careful handling of boundaries, linear diffusion can cause nearby objects to merge [S2, unofficial].

## Background
The Laplacian of Gaussian (LoG) is a classic edge and blob detection operator in image and volume processing. In volume rendering, it is used for scale-based transfer functions to highlight features of specific sizes [S2, unofficial]. The method was adapted for 3D volumes to provide a scale abstraction, allowing users to explore data based on feature size rather than just intensity. The normalized Laplacian helps in detecting blobs of varying sizes by compensating for the decay of the second derivative with scale [S2, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
