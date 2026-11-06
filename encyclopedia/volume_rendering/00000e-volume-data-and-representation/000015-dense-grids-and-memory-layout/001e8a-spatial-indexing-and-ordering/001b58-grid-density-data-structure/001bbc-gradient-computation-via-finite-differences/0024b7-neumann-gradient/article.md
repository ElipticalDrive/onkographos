# Neumann Gradient

Also called: 4D Linear Regression Gradient Estimation, Neumann Gradient Estimation, Regression

## Overview
The Neumann Gradient is a high-quality gradient estimation method for volume data that approximates the local density function using a 3D regression hyperplane fitted to 26 neighboring voxels [S1, unofficial][S2]. It produces nearly isotropic gradients with significantly reduced fringing artifacts compared to standard finite difference methods [S1, unofficial][S2]. Unlike [Gradient Map Generation](1204-gradient-map-generation.md) or [Analytic Gradient Computation](2167-analytic-gradient-computation.md), it relies on weighted linear regression over a cubic neighborhood to minimize approximation error, resulting in smoother visual appearance at the cost of higher computational complexity [S2][S3, unofficial].

## How it works
The method estimates the gradient vector $\nabla_f = [A, B, C]^T$ for a central voxel by fitting a plane $f(x,y,z) = Ax + By + Cz + D$ to the values of the 26 surrounding voxels in a $3 \times 3 \times 3$ neighborhood, excluding the center voxel itself [S2]. The fitting minimizes the weighted squared error, where weights are inversely proportional to the distance from the center [S2].

### Inputs and Outputs
*   **Input**: Scalar volume data $V$. For a target voxel at grid coordinates $(x_0, y_0, z_0)$, the input is the set of 26 neighbor values $v_k$ at relative coordinates $(x_k, y_k, z_k)$, where $k \in \{0, \dots, 25\}$ indexes the neighbors. The center voxel is at $(0,0,0)$.
*   **Output**: Gradient vector $\nabla_f = [A, B, C]^T$ and optionally the filtered density value $D$ [S2].

### Data Structures
*   **Neighbor Coordinates**: The 26 neighbors occupy all integer offsets $(dx, dy, dz)$ within $[-1, 1]^3$ except $(0,0,0)$.
*   **Weights**: A scalar weight $w_k$ for each neighbor $k$.

### Steps
1.  **Iterate** over the 26 neighbors $k$ of the current voxel.
2.  **Calculate Distance**: Compute the Euclidean distance $d_k$ from the center to neighbor $k$:
    $$ d_k = \sqrt{x_k^2 + y_k^2 + z_k^2} $$
3.  **Calculate Weight**: Compute the weight $w_k$ for each neighbor:
    $$ w_k = \frac{1}{d_k} $$
    [S2]
4.  **Compute Normalization Constants**: Calculate the constants $w_A, w_B, w_C, w_D$ required to solve the regression system. Note that $w_A$ and $w_C$ are constant for all voxels, while $w_B$ and $w_D$ depend on the geometry of the neighborhood (which is constant for a regular grid) [S2]:
    $$ w_A = \frac{1}{26} $$
    $$ w_B = \frac{1}{\sum_{k=0}^{25} w_k x_k^2} $$
    $$ w_C = \frac{1}{26} $$
    $$ w_D = \frac{1}{\sum_{k=0}^{25} w_k z_k^2} $$
    [S2]
    *Note: The source [S2] defines $w_B$ and $w_D$ using sums over $x_k^2$ and $z_k^2$ respectively. Due to the symmetry of the cubic neighborhood, $\sum w_k x_k^2 = \sum w_k y_k^2 = \sum w_k z_k^2$. Thus $w_B = w_D$.*
5.  **Compute Gradient Components**: Calculate $A, B, C$ using the weighted sums of the neighbor values $v_k$ and their coordinates:
    $$ A = w_A \sum_{k=0}^{25} w_k v_k x_k $$
    $$ B = w_B \sum_{k=0}^{25} w_k v_k y_k $$
    $$ C = w_C \sum_{k=0}^{25} w_k v_k z_k $$
    [S2]
6.  **Compute Filtered Density (Optional)**: If the filtered density value is needed for rendering (e.g., for opacity transfer), compute $D$:
    $$ D = w_D \sum_{k=0}^{25} w_k v_k $$
    [S2]
    *Note: Source [S1, unofficial] provides an equivalent separable filter mask formulation for the gradient and density. The matrix formulation in [S1, unofficial] yields the same result as the regression formulation in [S2] but is computationally structured as a 3D convolution. The weights in [S1, unofficial] are normalized by 52.*

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_neumann_gradient(volume, x, y, z):
    """
    Computes the Neumann gradient for the voxel at (x, y, z).
    Assumes volume is padded or boundary checks are handled externally.
    """
    # Precomputed constants for a 3x3x3 neighborhood excluding center
    # Neighbors are at offsets (dx, dy, dz) in [-1, 1]^3 \ {(0,0,0)}
    
    # 1. Define neighbor offsets and distances
    neighbors = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                dist_sq = dx*dx + dy*dy + dz*dz
                dist = sqrt(dist_sq)
                weight = 1.0 / dist
                neighbors.append({
                    'dx': dx, 'dy': dy, 'dz': dz,
                    'w': weight
                })
    
    # 2. Precompute normalization factors (constant for regular grid)
    # Sum of w_k * x_k^2 for x-direction
    sum_wx2 = sum(n['w'] * n['dx']**2 for n in neighbors)
    # Sum of w_k * z_k^2 for z-direction (symmetric to x)
    sum_wz2 = sum(n['w'] * n['dz']**2 for n in neighbors)
    
    w_A = 1.0 / 26.0
    w_B = 1.0 / sum_wx2
    w_C = 1.0 / 26.0
    w_D = 1.0 / sum_wz2
    
    # 3. Accumulate weighted sums
    sum_wvx = 0.0
    sum_wvy = 0.0
    sum_wvz = 0.0
    sum_wv = 0.0
    
    for n in neighbors:
        # Get voxel value at neighbor position
        # Note: In a real implementation, bounds checking is required
        val = volume[x + n['dx'], y + n['dy'], z + n['dz']]
        
        sum_wvx += n['w'] * val * n['dx']
        sum_wvy += n['w'] * val * n['dy']
        sum_wvz += n['w'] * val * n['dz']
        sum_wv += n['w'] * val
        
    # 4. Compute gradient components
    grad_x = w_A * sum_wvx
    grad_y = w_B * sum_wvy
    grad_z = w_C * sum_wvz
    
    # 5. Compute filtered density (optional)
    filtered_val = w_D * sum_wv
    
    return (grad_x, grad_y, grad_z), filtered_val
```

### Edge Cases and Pitfalls
*   **Boundary Conditions**: The method requires 26 neighbors. Voxels on the boundary of the volume do not have a full neighborhood. [G] Standard practice is to clamp coordinates to the volume bounds or use a smaller, asymmetric stencil for boundary voxels, though this breaks isotropy.
*   **Computational Cost**: The method involves 26 multiplications and additions per component, plus distance calculations. It is considerably more expensive than central differences [S1, unofficial].
*   **Smoothing Artifacts**: Using the filtered density value $D$ for rendering causes fading of small details, which may be undesirable in applications requiring high-frequency detail preservation, such as medical diagnosis [S1, unofficial].
*   **Precision**: The weights $w_k$ involve division by distance. For neighbors at distance 1, $w_k=1$. For distance $\sqrt{2}$, $w_k=1/\sqrt{2}$. For distance $\sqrt{3}$, $w_k=1/\sqrt{3}$. Floating point precision should be maintained to avoid bias in the regression.

## Subtopics

### Local 3D Regression Hyperplane Gradient Estimation
This is the formal name for the Neumann Gradient method as described in [S3, unofficial]. It is distinguished from Central Differences by estimating the density function via a local 3D regression hyperplane rather than using simple linear differences between neighboring voxels. This results in higher approximation quality and computational effort [S3, unofficial]. It is chosen when visual smoothness and isotropy are prioritized over performance and high-frequency detail preservation.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel, but with a high constant factor due to 26 neighbor accesses and weighted sums. It is significantly slower than central differences [S1, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory per voxel if computed on-the-fly. If precomputed, it requires storing the gradient vector (3 floats) and optionally the filtered density (1 float) per voxel.
*   **Performance**: On a $16 \times 16 \times 16$ volume, it produces smoother gradient magnitudes than central differences or polynomial interpolation [S2]. However, the computational cost is considered "considerably high" [S1, unofficial].
*   **Strengths**: Nearly isotropic gradients, reduced fringing artifacts, smoother appearance in regions of rapid gradient direction change [S1, unofficial][S2].
*   **Limitations**: High computational cost, loss of small details if filtered density is used, boundary handling issues [S1, unofficial].
*   **When to use**: High-quality offline rendering or real-time rendering on powerful hardware where visual smoothness is critical and performance is not the primary constraint [S1, unofficial].
*   **When not to use**: Real-time medical applications where small details must be preserved and performance is limited [S1, unofficial].

## Background
The Neumann Gradient was proposed by Neumann et al. [S1, unofficial][S2]. It is based on the theoretical framework of linear regression [S1, unofficial]. The method is also referred to as "4D linear regression" in some literature [S2]. It is used in high-quality volume rendering systems to improve the visual appearance of gradients, particularly for lighting calculations [S1, unofficial].

## Sources
[S1] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S2] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S3] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
