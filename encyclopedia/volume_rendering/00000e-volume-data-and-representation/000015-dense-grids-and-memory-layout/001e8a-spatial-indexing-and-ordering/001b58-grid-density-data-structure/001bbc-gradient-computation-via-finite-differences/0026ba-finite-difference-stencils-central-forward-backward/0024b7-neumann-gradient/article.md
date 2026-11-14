# Neumann Gradient

Also called: 3D hyperplane approximation, 4D Linear Regression Gradient Estimation, Gradient estimation based on linear regression, Linear Regression Gradient Estimation, Neumann Gradient Estimation

## Overview
The Neumann Gradient is a gradient estimation method for volume data that approximates the local density function using a 3D regression hyperplane fitted to 26 neighboring voxels [S1, S3]. It solves the problem of staircase artifacts and anisotropic errors inherent in simple finite difference stencils (like [Central Difference Stencils](9968-central-difference-stencils.md)) by providing nearly isotropic gradients with reduced fringing artifacts [S2, unofficial]. Unlike [Central Difference Stencils](9968-central-difference-stencils.md) which rely on direct linear differences between immediate neighbors, Neumann Gradient minimizes a weighted mean-square error over a larger neighborhood, resulting in smoother specular effects and better handling of rapid gradient direction changes [S1, S3].

## How it works
The method estimates the gradient $\nabla V = (G_x, G_y, G_z)$ at a voxel center $\mathbf{p} = (x_0, y_0, z_0)$ by fitting a plane $f(x,y,z) = Ax + By + Cz + D$ to the density values of the 26 neighbors. The coefficients $A, B, C$ correspond to the partial derivatives $\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z}$ [S3].

### Inputs and Outputs
*   **Input:** A 3D scalar field $V$ (density/opacity). For a target voxel at integer coordinates $(x_0, y_0, z_0)$, the method requires density values $v_k$ at 26 neighboring positions $\mathbf{p}_k = (x_k, y_k, z_k)$ surrounding the center. The center voxel itself is excluded from the regression set.
*   **Output:** A 3D gradient vector $\nabla V = [A, B, C]^T$.

### Data Structures
The implementation requires accessing a 3D grid. The 26 neighbors are defined by all integer offsets $(dx, dy, dz)$ where $dx, dy, dz \in \{-1, 0, 1\}$ excluding $(0,0,0)$.

### Parameters
*   **Voxel Spacing:** Assumed isotropic, $h=1$ for integer indexing. If physical units are used, the final gradient components must be scaled by $1/h$ [S3, G].
*   **Weighting Function:** A spherically symmetric weight $w_k$ is applied to each neighbor based on its distance $d_k$ from the center [S3].
    *   $d_k = \sqrt{x_k^2 + y_k^2 + z_k^2}$ (Euclidean distance in voxel units).
    *   $w_k = 1 / d_k$ [S3].
    *   *Note:* [S2, unofficial] presents a separable filter mask formulation that implicitly handles weighting through convolution kernels, but the explicit linear regression formulation [S3] is the primary definition.

### Formulas
The plane equation coefficients are calculated as [S3]:
$$
A = w_A \sum_{k=0}^{26} w_k v_k x_k
$$
$$
B = w_B \sum_{k=0}^{26} w_k v_k y_k
$$
$$
C = w_C \sum_{k=0}^{26} w_k v_k z_k
$$
$$
D = w_D \sum_{k=0}^{26} w_k v_k
$$

Where the normalization constants are [S3]:
$$
w_A = \frac{1}{26}
$$
$$
w_B = \frac{1}{\sum_{k=0}^{26} w_k x_k^2}
$$
$$
w_C = \frac{1}{26}
$$
$$
w_D = \frac{1}{\sum_{k=0}^{26} w_k z_k^2}
$$

The gradient is $\nabla V = [A, B, C]^T$.

*Alternative Separable Filter Formulation:*
[S2, unofficial] provides equivalent 3x3x3 convolution kernels for $\nabla_x, \nabla_y, \nabla_z$ scaled by $1/52$. These kernels are derived from the regression weights.
For $\nabla_x$:
$$
K_x = \begin{pmatrix} 
-2 & 0 & 2 \\
-3 & 0 & 3 \\
-2 & 0 & 2 
\end{pmatrix} \text{ (slice } y=-1) \cup \begin{pmatrix} 
-3 & 0 & 3 \\
-6 & 0 & 6 \\
-3 & 0 & 3 
\end{pmatrix} \text{ (slice } y=0) \cup \begin{pmatrix} 
-2 & 0 & 2 \\
-3 & 0 & 3 \\
-2 & 0 & 2 
\end{pmatrix} \text{ (slice } y=1)
$$
*Correction on Source [S2, unofficial] Matrix:* The source [S2, unofficial] displays $\nabla_x$ as a 3x3x3 structure. The middle slice (y=0) has weights $\pm 3, \pm 6$. The outer slices have $\pm 2, \pm 3$. The center column (x=0) is zero.
For $\nabla_y$:
$$
K_y = \begin{pmatrix} 
2 & 3 & 2 \\
0 & 0 & 0 \\
-2 & -3 & -2 
\end{pmatrix} \text{ (slice } z=-1) \cup \begin{pmatrix} 
3 & 6 & 3 \\
0 & 0 & 0 \\
-3 & -6 & -3 
\end{pmatrix} \text{ (slice } z=0) \cup \begin{pmatrix} 
2 & 3 & 2 \\
0 & 0 & 0 \\
-2 & -3 & -2 
\end{pmatrix} \text{ (slice } z=1)
$$
For $\nabla_z$:
$$
K_z = \begin{pmatrix} 
-2 & -3 & -2 \\
-3 & -6 & -3 \\
-2 & -3 & -2 
\end{pmatrix} \text{ (slice } x=-1) \cup \begin{pmatrix} 
0 & 0 & 0 \\
0 & 0 & 0 \\
0 & 0 & 0 
\end{pmatrix} \text{ (slice } x=0) \cup \begin{pmatrix} 
2 & 3 & 2 \\
3 & 6 & 3 \\
2 & 3 & 2 
\end{pmatrix} \text{ (slice } x=1)
$$
All components are divided by 52 [S2, unofficial].

### Steps
1.  **Identify Neighbors:** For the target voxel at $(x_0, y_0, z_0)$, identify the 26 neighbors at $(x_0+dx, y_0+dy, z_0+dz)$ for $dx,dy,dz \in \{-1,0,1\} \setminus \{(0,0,0)\}$.
2.  **Handle Boundaries:** If any neighbor is outside the volume bounds, apply boundary handling (e.g., clamping or zero-padding) [G]. The source [S2, unofficial] notes that boundary handling is critical.
3.  **Compute Weights:** For each neighbor $k$, calculate distance $d_k = \sqrt{dx^2 + dy^2 + dz^2}$ and weight $w_k = 1/d_k$.
4.  **Compute Sums:** Calculate the sums required for $A, B, C$ (and optionally $D$ for filtering):
    *   $S_{Ax} = \sum w_k v_k dx$
    *   $S_{By} = \sum w_k v_k dy$
    *   $S_{Cz} = \sum w_k v_k dz$
    *   $S_{xx} = \sum w_k dx^2$
    *   $S_{zz} = \sum w_k dz^2$
5.  **Calculate Coefficients:**
    *   $A = S_{Ax} / 26$
    *   $B = S_{By} / S_{xx}$
    *   $C = S_{Cz} / 26$
6.  **Scale:** If voxel spacing $h \neq 1$, divide $A, B, C$ by $h$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neumann_gradient(volume, x, y, z, h=1.0):
    """
    Computes the Neumann gradient at voxel (x, y, z).
    volume: 3D array of density values.
    x, y, z: Integer coordinates of the center voxel.
    h: Voxel spacing (default 1.0).
    """
    # Constants
    w_A = 1.0 / 26.0
    w_C = 1.0 / 26.0
    
    # Initialize sums
    sum_A = 0.0
    sum_B_num = 0.0
    sum_B_den = 0.0
    sum_C = 0.0
    sum_D_num = 0.0
    sum_D_den = 0.0
    
    # Iterate over 3x3x3 neighborhood excluding center
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                
                # Neighbor coordinates
                nx = x + dx
                ny = y + dy
                nz = z + dz
                
                # Boundary handling: clamp to valid range
                nx = max(0, min(nx, volume.shape[0] - 1))
                ny = max(0, min(ny, volume.shape[1] - 1))
                nz = max(0, min(nz, volume.shape[2] - 1))
                
                # Get density value
                v_k = volume[nx, ny, nz]
                
                # Distance and weight
                d_k_sq = dx*dx + dy*dy + dz*dz
                d_k = d_k_sq ** 0.5
                w_k = 1.0 / d_k
                
                # Accumulate sums for A, B, C, D
                # A = w_A * sum(w_k * v_k * x_k) where x_k is relative coord dx
                sum_A += w_k * v_k * dx
                
                # B = w_B * sum(w_k * v_k * y_k) where y_k is relative coord dy
                # w_B = 1 / sum(w_k * y_k^2)
                sum_B_num += w_k * v_k * dy
                sum_B_den += w_k * dy * dy
                
                # C = w_C * sum(w_k * v_k * z_k) where z_k is relative coord dz
                sum_C += w_k * v_k * dz
                
                # D is optional (for filtered density), but calculated similarly
                sum_D_num += w_k * v_k
                sum_D_den += w_k * dz * dz # Note: Source [S3] uses z_k^2 for D denominator? 
                                           # Actually [S3] says w_D = 1 / sum(w_k z_k^2). 
                                           # This seems specific to the D term derivation in that paper.
                                           # We compute it for completeness if D is needed.

    # Calculate coefficients
    # A and C use fixed weight 1/26
    A = w_A * sum_A
    C = w_C * sum_C
    
    # B uses variable weight
    if sum_B_den == 0:
        B = 0.0
    else:
        B = sum_B_num / sum_B_den
        
    # D is not part of the gradient, but often computed alongside
    # D = (1.0 / sum_D_den) * sum_D_num if needed for filtering

    # Scale by voxel spacing
    if h != 1.0:
        A /= h
        B /= h
        C /= h
        
    return (A, B, C)
```

### Edge Cases and Pitfalls
*   **Boundary Conditions:** The stencil requires 26 neighbors. Voxels on the volume boundary lack neighbors. Clamping (repeating edge values) is a common [G] strategy, but may introduce bias. Zero-padding is another option.
*   **Singularity:** If all neighbors have the same coordinate in a dimension (e.g., all $dy=0$), the denominator for $B$ becomes zero. This is impossible in a 3D grid with a full 3x3x3 neighborhood, but possible if the neighborhood is truncated.
*   **Precision:** The method involves division by distances and sums of squares. Floating-point precision issues are minimal for standard medical imaging resolutions but should be monitored for very large volumes or extreme aspect ratios.
*   **Smoothing:** The method acts as a low-pass filter. Small details may be faded [S2, unofficial].

## Subtopics

### [Local 3D Regression Hyperplane Gradient Estimation](8300-local-3d-regression-hyperplane-gradient-estimation.md)
A high-quality variant that fits a local 3D regression hyperplane to the density function. It differs from Central Differences by using regression over a neighborhood rather than simple linear differences, offering higher approximation quality at the cost of increased computational effort.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel, but with a high constant factor due to 26 neighbor accesses, distance calculations, and weighted sums. It is significantly slower than [Central Difference Stencils](9968-central-difference-stencils.md) which require only 6 accesses and simple subtractions.
*   **Memory Complexity:** $O(1)$ auxiliary memory per voxel.
*   **Performance:** Considerably high computational costs [S2, unofficial]. Suitable for offline or high-quality rendering, but challenging for real-time applications on standard hardware without optimization (e.g., separable filter implementation) [S2, unofficial].
*   **Strengths:** Nearly isotropic gradients, reduced fringing artifacts, smoother appearance in regions of rapid gradient change [S2, unofficial][S3].
*   **Limitations:** Fades small details, which may be unacceptable in medical applications requiring high-frequency detail preservation [S2, unofficial].
*   **When to Use:** High-quality non-photorealistic rendering, scientific visualization where smooth normals are preferred over sharp edges, and when isotropy is critical.
*   **When Not to Use:** Real-time medical diagnosis requiring fine detail, or when computational resources are severely constrained.

## Background
The Neumann Gradient was proposed by Neumann et al. [S3] as a "4D linear regression" method for gradient estimation. It approximates the density function in a local neighborhood with a 3D regression hyperplane, minimizing error from 26 neighboring voxels weighted by a spherically symmetric function [S3]. It was developed to address the staircase artifacts and anisotropy of central differences in volume rendering, particularly for non-photorealistic rendering and surface extraction [S1, S3]. The method is cited in works on volume illustration and non-photorealistic rendering [S1, S3].

## Sources
[S1] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
[S2] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S3] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S4] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
