# Half-Space Field Generation

Also called: Binary Half-Volume, Level Set Domain Definition, Level Set Masking, Step Function Field

## Overview
Half-Space Field Generation defines a volumetric domain by partitioning space into two disjoint regions (typically "inside" and "outside") using a binary scalar field or a level set threshold. It solves the problem of restricting simulation or rendering computations to a specific geometric subset, effectively ignoring empty space or non-fluid regions. Unlike smooth implicit primitives that blend continuously, this method produces a sharp discontinuity, making it distinct from [Metaball Field Evaluation](1017-metaball-field-evaluation.md) and [Soft Metablobby SDF](2753-soft-metablobby-sdf.md). It is primarily used to mask out irrelevant voxels in fluid simulations or ray marching to save bandwidth and compute time.

## How it works
**Inputs and Outputs:**
- Input: A 3D grid of dimensions $(X, Y, Z)$ and a scalar field $\phi(\mathbf{x})$ (the level set) or a binary condition.
- Output: A binary mask field $M(\mathbf{x}) \in \{0, 1\}$ or a modified physical field where values outside the domain are zeroed.

**Data Structures:**
- A 3D array `float3d[X][Y][Z]` or `int3d[X][Y][Z]` storing the field values.
- In GPU implementations, this is often stored as a 3D texture or a flattened 1D texture array.

**Steps:**
1. **Initialize** the field array with default values (e.g., 0 or background noise).
2. **Evaluate** the level set function $\phi(\mathbf{x})$ for each voxel center $\mathbf{x}$.
3. **Apply Threshold:** Determine the binary state based on the sign of $\phi(\mathbf{x})$.
   - If $\phi(\mathbf{x}) < 0$, the point is inside the domain (fluid/solid).
   - If $\phi(\mathbf{x}) \ge 0$, the point is outside (air/empty).
4. **Mask/Zero Out:**
   - For simulation: Set pressure, velocity, or density to 0 for all $\mathbf{x}$ where $\phi(\mathbf{x}) \ge 0$.
   - For binary generation: Set $M(\mathbf{x}) = 1$ if $\phi(\mathbf{x}) < 0$, else $M(\mathbf{x}) = 0$.
5. **Compute Forces:** Apply external forces (e.g., gravity) only to voxels where $M(\mathbf{x}) = 1$.

**Formulas:**
The binary mask $M(\mathbf{x})$ is defined as:
$$
M(\mathbf{x}) = \begin{cases} 
1 & \text{if } \phi(\mathbf{x}) < \theta \\
0 & \text{if } \phi(\mathbf{x}) \ge \theta 
\end{cases}
$$
Where $\theta$ is the threshold, typically $0$ for signed distance fields or level sets [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_half_space_field(phi_grid, threshold=0.0):
    """
    Generates a binary mask from a level set grid.
    phi_grid: 3D array of floats, shape (X, Y, Z)
    threshold: float, typically 0.0
    Returns: 3D array of ints (0 or 1)
    """
    X, Y, Z = phi_grid.shape
    mask = np.zeros((X, Y, Z), dtype=np.int32)
    
    for z in range(Z):
        for y in range(Y):
            for x in range(X):
                # Evaluate level set at voxel center
                val = phi_grid[x, y, z]
                
                # Apply binary threshold
                if val < threshold:
                    mask[x, y, z] = 1
                else:
                    mask[x, y, z] = 0
                    
    return mask

def apply_domain_masking(field_grid, mask):
    """
    Zeros out values in a physical field based on the binary mask.
    Used to ignore air cells in fluid simulation.
    """
    for z in range(field_grid.shape[2]):
        for y in range(field_grid.shape[1]):
            for x in range(field_grid.shape[0]):
                if mask[x, y, z] == 0:
                    field_grid[x, y, z] = 0.0
    return field_grid
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point comparisons near the threshold $\theta=0$ can cause jitter if the level set is not reinitialized or smoothed [S2, unofficial].
- **Boundaries:** Ensure the grid extends sufficiently beyond the implicit surface to avoid clipping the domain.
- **Performance:** Iterating over all voxels to check the mask is expensive if the domain is sparse.

**Subtopics:**
- [Level Set Masking](2798-implicit-surface-definition.md): Using the sign of the level set directly in shaders to cull fragments.
- [Sparse Domain Definition](13-procedural-and-implicit-volumes.md): Using sparse structures (like OpenVDB) to store only the non-zero mask regions, avoiding the memory cost of dense binary grids.

## Operating characteristics
- **Time Complexity:** $O(X \cdot Y \cdot Z)$ for generating the mask or applying it to a dense grid.
- **Memory Complexity:** $O(X \cdot Y \cdot Z)$ for the dense mask. Can be reduced to $O(N_{active})$ using sparse structures [S2, unofficial].
- **Performance:** On GPUs, branching on the mask value (`if phi >= 0`) can cause warp divergence if dynamic flow control is not supported. Alternatives include using z-cull to eliminate cells or using sparse data structures [S2, unofficial].
- **Strengths:** Simple to implement; effectively reduces simulation domain size; computationally cheap per voxel.
- **Limitations:** Sharp discontinuities can cause aliasing in rendering; dense storage is memory-intensive for large domains with small objects.

## Background
The method originates from fluid simulation techniques where the air is assumed to have negligible effect on the liquid. By defining the fluid domain via a level set $\phi$, simulations can ignore cells where $\phi \ge 0$. This approach was detailed in *GPU Gems 3* for real-time 3D fluid simulation [S2, unofficial]. It is also used in procedural voxel generation to define binary volumes [S1, unofficial].

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
