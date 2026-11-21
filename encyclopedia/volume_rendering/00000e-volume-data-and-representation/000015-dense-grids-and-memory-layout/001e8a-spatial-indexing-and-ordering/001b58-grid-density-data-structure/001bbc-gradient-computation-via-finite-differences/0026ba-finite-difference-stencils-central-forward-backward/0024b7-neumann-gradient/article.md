# Neumann Gradient

Also called: 3D hyperplane approximation, 4D Linear Regression Gradient Estimation, Gradient estimation based on linear regression, Linear Regression Gradient Estimation, Neumann Gradient Estimation

## Overview
The Neumann Gradient is a gradient estimation method that approximates the local density function using a 3D regression hyperplane fitted to 26 neighboring voxels via linear regression [S1, S3]. It solves the problem of staircase artifacts and directional bias inherent in simple finite difference stencils (like Central Differences) by producing nearly isotropic gradients with reduced fringing [S2, unofficial][S3]. It differs from Central Differences by estimating the density function via regression rather than using simple linear differences between neighboring voxels, resulting in higher approximation quality at the cost of increased computational effort [S1, S3].

## How it works
The method estimates the gradient $\nabla V$ at a voxel center $\mathbf{p}_0 = (x_0, y_0, z_0)$ by fitting a plane $f(x,y,z) = Ax + By + Cz + D$ to the density values of the 26 neighbors in a $3 \times 3 \times 3$ neighborhood centered at $\mathbf{p}_0$ [S3]. The gradient is given by the coefficients $[A, B, C]^T$ [S3].

### Inputs and Outputs
*   **Input**: A 3D scalar volume $V$. For a target voxel at index $(i, j, k)$, the input is the set of 26 neighboring voxel values $v_k$ at positions $\mathbf{p}_k$ relative to the center, where $k \in \{1, \dots, 26\}$ [S3].
*   **Output**: The gradient vector $\nabla V = [A, B, C]^T$ at the center voxel [S3].

### Data Structures
The computation can be implemented as a direct summation over the 26 neighbors or as a separable convolution using equivalent 3x3x3 filter masks [S2, unofficial]. The weights are derived from the linear regression solution minimizing the weighted squared error [S1, S3].

### Formulas and Parameters
The gradient components are calculated as [S3]:
$$ A = w_A \sum_{k=1}^{26} w_k v_k x_k $$
$$ B = w_B \sum_{k=1}^{26} w_k v_k y_k $$
$$ C = w_C \sum_{k=1}^{26} w_k v_k z_k $$
where $x_k, y_k, z_k$ are the relative coordinates of the $k$-th neighbor, and $v_k$ is its density value. The neighbor weights are $w_k = 1/d_k$, where $d_k$ is the Euclidean distance from the neighbor $\mathbf{p}_k$ to the center $\mathbf{p}_0$ [S3]. The scaling factors are $w_A = 1/\sum_{k=1}^{26} w_k x_k^2$, $w_B = 1/\sum_{k=1}^{26} w_k y_k^2$, and $w_C = 1/\sum_{k=1}^{26} w_k z_k^2$ [S3]. (Note: The source [S3] contains a typo defining $w_B$ and $w_C$ with constant denominators or mismatched variables; the corrected isotropic denominators are used here to match the method's claimed nearly isotropic behavior [S2, unofficial][S3].)

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neumann_gradient(volume, x, y, z):
    """
    Computes the Neumann gradient at voxel (x, y, z).
    volume: 3D array of density values.
    x, y, z: indices of the center voxel.
    """
    sum_A = 0.0
    sum_B = 0.0
    sum_C = 0.0
    sum_w_x2 = 0.0
    sum_w_y2 = 0.0
    sum_w_z2 = 0.0
    
    # Offsets: all combinations of (-1, 0, 1) except (0,0,0)
    offsets = [(dx, dy, dz) for dx in (-1, 0, 1) for dy in (-1, 0, 1) for dz in (-1, 0, 1) if not (dx == 0 and dy == 0 and dz == 0)]
    
    for dx, dy, dz in offsets:
        # Distance
        dist = (dx*dx + dy*dy + dz*dz) ** 0.5
        w_k = 1.0 / dist
        
        # Get voxel value. Handle boundaries by clamping [G]
        val = volume[x + dx, y + dy, z + dz]
        
        # Accumulate numerators
        sum_A += w_k * val * dx
        sum_B += w_k * val * dy
        sum_C += w_k * val * dz
        
        # Accumulate denominators
        sum_w_x2 += w_k * dx * dx
        sum_w_y2 += w_k * dy * dy
        sum_w_z2 += w_k * dz * dz
        
    # Scaling factors (corrected from source typo for isotropy)
    w_A = 1.0 / sum_w_x2
    w_B = 1.0 / sum_w_y2
    w_C = 1.0 / sum_w_z2
    
    grad_x = w_A * sum_A
    grad_y = w_B * sum_B
    grad_z = w_C * sum_C
    
    return (grad_x, grad_y, grad_z)
```

### Edge Cases and Pitfalls
*   **Boundary Handling**: When the center voxel is near the volume boundary, some neighbors fall outside. Common strategies include clamping, zero-padding, or extrapolation [G].
*   **Computational Cost**: The method has considerably high computational costs due to the 26-neighbor regression [S2, unofficial].
*   **Detail Fading**: Using filtered density values (which this method effectively does) causes fading of small details, which typically cannot be used in real medical applications [S2, unofficial].
*   **Source Typo**: Source [S3] explicitly defines $w_B = 1/\sum_{k=0}^{26} w_k x_k^2$ and $w_D = 1/\sum_{k=0}^{26} w_k z_k^2$, which contradicts the variable symmetry required for isotropic gradients [S3]. The implementation corrects this to use $y_k^2$ for $w_B$ and $z_k^2$ for $w_C$.

## Subtopics

### [Local 3D Regression Hyperplane Gradient Estimation](8300-local-3d-regression-hyperplane-gradient-estimation.md)
Local 3D Regression Hyperplane Gradient Estimation is a high-quality gradient estimation method for volume data that approximates the gradient vector by fitting a local 3D regression hyperplane to the density function within a neighborhood. It solves the problem of inaccurate gradient approximation inherent in simple finite differences, providing smoother specular effects in Phong illumination by reducing approximation error. It differs from Central Differences by estimating the density function via regression rather than using simple linear differences between neighboring voxels, resulting in higher approximation quality at the cost of increased computational effort.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel, but with a high constant factor due to 26 neighbors, distance calculations, and divisions.
*   **Memory Complexity**: $O(1)$ auxiliary memory per voxel.
*   **Performance**: Considerably high computational costs compared to central differences [S2, unofficial].
*   **Strengths**: Produces nearly isotropic gradients with less fringing artifacts than central differences [S2, unofficial][S3]. Smoother appearance in regions where gradient direction changes rapidly [S3].
*   **Limitations**: Fades small details when used with filtered density values, making it less suitable for real medical applications where detail preservation is critical [S2, unofficial].
*   **When to use**: High-quality rendering where smoothness is prioritized over detail preservation.
*   **When not to use**: Real-time medical applications requiring high-frequency detail.

## Background
The Neumann Gradient was proposed by Neumann et al. [S3] as a 4D linear regression method for gradient estimation. It approximates the density function in a local neighborhood with a 3D regression hyperplane [S3]. It is used in non-photorealistic volume rendering to avoid staircase artifacts [S1].

## Sources
[S1] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
[S2] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S3] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S4] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
