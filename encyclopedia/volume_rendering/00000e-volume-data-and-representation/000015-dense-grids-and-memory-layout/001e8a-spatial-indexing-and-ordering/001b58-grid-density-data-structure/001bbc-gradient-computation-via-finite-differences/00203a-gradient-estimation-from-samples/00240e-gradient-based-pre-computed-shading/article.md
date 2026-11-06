# Gradient-based Pre-computed Shading

Also called: ComputeShading, Precomputed Lambertian Shading, View-independent voxel shading

## Overview
Gradient-based Pre-computed Shading is a preprocessing technique that calculates shaded colors for each voxel in a volume dataset using a view-independent Lambertian shading model [S1, S2, unofficial]. It solves the performance bottleneck of computing surface normals and lighting interactions per-ray during rendering by storing the result in a separate "shading volume" [S2, unofficial]. Unlike [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md), which provides raw gradient vectors for general use, this method specifically applies a diffuse lighting model to produce final pixel-intensity contributions [S1, S2, unofficial]. It differs from [Gradient-Based Surface Shading](5648-gradient-based-surface-shading.md) by operating on the dense voxel grid directly rather than extracting an isosurface mesh [S1, S2, unofficial].

## How it works
**Inputs:**
*   Scalar field volume $CT$ of dimensions $COLS \times ROWS \times SLCS$ (x, y, z) [S2, unofficial].
*   Light direction vector $\mathbf{L}$ [S1, S2, unofficial].
*   Diffuse reflectivity coefficient $k_d$ [S2, unofficial].
*   Light intensity $I_p$ [S2, unofficial].
*   Epsilon threshold $\epsilon$ (typically $7.25$) to handle flat regions [S2, unofficial].

**Outputs:**
*   Shading volume $SHADING$ of the same dimensions as $CT$, storing unsigned char values [S2, unofficial].

**Data Structures:**
*   $CT[z][y][x]$: Float array representing the scalar field (e.g., CT density) [S2, unofficial].
*   $SHADING[z][y][x]$: Unsigned char array storing the pre-computed shading value [S2, unofficial].

**Steps:**
1.  Iterate through the volume grid for $z$ from $1$ to $SLCS-2$, $y$ from $1$ to $ROWS-2$, and $x$ from $1$ to $COLS-2$, skipping boundary layers [S2, unofficial].
2.  Compute partial derivatives using central finite differences [S1, S2, unofficial]:
    $$ d_x = \frac{CT[z][y][x+1] - CT[z][y][x-1]}{2.0} $$
    $$ d_y = \frac{CT[z][y+1][x] - CT[z][y-1][x]}{2.0} $$
    $$ d_z = \frac{CT[z+1][y][x] - CT[z-1][y][x]}{2.0} $$
3.  Compute the gradient magnitude $D$ [S2, unofficial]:
    $$ D = \sqrt{d_x^2 + d_y^2 + d_z^2} $$
4.  Check for degenerate gradients. If $D < \epsilon$, set $SHADING[z][y][x] = 0$ [S2, unofficial].
5.  Otherwise, compute the normalized normal vector $\mathbf{N}$ and the diffuse shading value [S2, unofficial]:
    $$ \mathbf{N} = \left( \frac{d_x}{D}, \frac{d_y}{D}, \frac{d_z}{D} \right) $$
    $$ S = I_p \cdot k_d \cdot (\mathbf{N} \cdot \hat{\mathbf{L}}) $$
    where $\hat{\mathbf{L}} = \frac{\mathbf{L}}{|\mathbf{L}|}$ is the normalized light direction [S2, unofficial].
6.  Store the result in the shading volume:
    $$ SHADING[z][y][x] = \text{unsigned char}(S) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_shading(CT, COLS, ROWS, SLCS, L, kd, Ip, epsilon=7.25):
    """
    Pre-computes shading for a volume grid using Lambertian model.
    
    Args:
        CT: 3D list/array of floats, shape (SLCS, ROWS, COLS).
        COLS, ROWS, SLCS: Dimensions of the volume.
        L: List/tuple of 3 floats, light direction vector.
        kd: Float, diffuse reflectivity.
        Ip: Float, light intensity.
        epsilon: Float, threshold for gradient magnitude.
        
    Returns:
        SHADING: 3D list of unsigned chars (0-255).
    """
    # Initialize shading volume with zeros
    SHADING = [[[0 for _ in range(COLS)] for _ in range(ROWS)] for _ in range(SLCS)]
    
    # Normalize light direction
    L_len = math.sqrt(L[0]**2 + L[1]**2 + L[2]**2)
    if L_len == 0:
        return SHADING
    L_hat = (L[0]/L_len, L[1]/L_len, L[2]/L_len)
    
    # Iterate over interior voxels, skipping boundaries
    for z in range(1, SLCS - 1):
        for y in range(1, ROWS - 1):
            for x in range(1, COLS - 1):
                # Central differences for partial derivatives
                dx = (CT[z][y][x+1] - CT[z][y][x-1]) / 2.0
                dy = (CT[z][y+1][x] - CT[z][y-1][x]) / 2.0
                dz = (CT[z+1][y][x] - CT[z-1][y][x]) / 2.0
                
                # Gradient magnitude
                D_sq = dx*dx + dy*dy + dz*dz
                D = math.sqrt(D_sq)
                
                # Handle flat regions/noise
                if D < epsilon:
                    SHADING[z][y][x] = 0
                else:
                    # Normalized normal vector
                    nx = dx / D
                    ny = dy / D
                    nz = dz / D
                    
                    # Diffuse shading: Ip * kd * (N . L_hat)
                    dot_product = nx * L_hat[0] + ny * L_hat[1] + nz * L_hat[2]
                    shading_val = Ip * kd * dot_product
                    
                    # Clamp and cast to unsigned char
                    # Note: Sources imply simple casting, but standard practice clamps to [0, 255]
                    # The source code uses (unsigned char)(...), which truncates.
                    # We assume the source handles negative values or expects positive shading.
                    # To be safe and implementable, we clamp.
                    if shading_val < 0:
                        shading_val = 0
                    elif shading_val > 255:
                        shading_val = 255
                        
                    SHADING[z][y][x] = int(shading_val)
                    
    return SHADING
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The algorithm skips the outermost layer of voxels ($x=0, x=COLS-1$, etc.) because central differences require neighbors on both sides [S2, unofficial].
*   **Flat Regions:** If the gradient magnitude $D$ is below $\epsilon$ (e.g., $7.25$), the normal is undefined or noisy. The method sets the shading to 0 to avoid division by zero and noise amplification [S2, unofficial].
*   **Data Types:** The output is stored as `unsigned char`, which may clip negative shading values if the light direction results in a negative dot product (back-facing surfaces) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the volume, as each interior voxel is processed once [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the additional shading volume [S2, unofficial].
*   **Performance:** Significantly reduces rendering time by moving normal computation and lighting calculations to a preprocessing step [S1, S2, unofficial].
*   **Strengths:** View-independent shading allows for fast ray casting without per-ray normal estimation [S1, S2, unofficial].
*   **Limitations:** Cannot handle view-dependent effects like specular highlights or environment reflections [S1, unofficial]. The fixed light direction means shading does not change with camera movement [S1, unofficial].

## Background
This method originates from early interactive volume rendering techniques for medical data visualization [S1, unofficial]. It was implemented in systems such as the one described by kkhese, where it was named `ComputeShading` [S2, unofficial]. The approach leverages the efficiency of lookup-table techniques and binary row representations to skip transparent cells during ray casting [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] kkhese/Volume-Rendering, https://github.com/kkhese/Volume-Rendering, fetched 2026-10-26 (unofficial)
