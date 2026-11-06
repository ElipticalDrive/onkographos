# Gradient-Based Surface Shading

Also called: Gradient Computation for Shading, Gradient Estimation, Gradient Shading, Gradient-based Volume Shading, Normal Computation for Shading, Normal Vector Computation, Surface Normal Estimation

## Overview
Gradient-based surface shading approximates the surface normal at a query point by computing the normalized gradient of the underlying scalar field [S1, unofficial]. This method enhances the perception of shape and boundaries between materials in volume-rendered images [S1, unofficial]. It differs from direct lighting models by deriving illumination cues from the data's intrinsic spatial variation rather than external light sources, though it is computationally more expensive than simple dot-product shading due to gradient evaluation [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar field $V(\mathbf{x})$ defined over a 3D domain, where $\mathbf{x} = (x, y, z)$ is the spatial coordinate [G].
*   **Output:** A surface normal vector $\mathbf{n}$ at $\mathbf{x}$, used for shading calculations (e.g., diffuse reflection) [S1, unofficial].

**Data Structures**
The scalar field is stored as a dense 3D grid [G]. To compute the gradient, the method requires access to neighboring voxel values [G]. If the field is stored as a 1D array of size $N_x \times N_y \times N_z$, the value at index $(i, j, k)$ is accessed as $V[i][j][k]$ [G].

**Algorithm Steps**
1.  **Evaluate Field Value:** Determine the scalar value $V(\mathbf{x})$ at the query point $\mathbf{x}$ using trilinear interpolation from the surrounding 8 voxels [G].
2.  **Compute Gradient Components:** Calculate the partial derivatives $\frac{\partial V}{\partial x}$, $\frac{\partial V}{\partial y}$, and $\frac{\partial V}{\partial z}$ using central finite differences [S1, unofficial]. For a grid with uniform spacing $\Delta$, the gradient $\nabla V$ at $\mathbf{x}$ is:
    $$
    \nabla V(\mathbf{x}) = \begin{bmatrix} \frac{V(x+\Delta, y, z) - V(x-\Delta, y, z)}{2\Delta} \\ \frac{V(x, y+\Delta, z) - V(x, y-\Delta, z)}{2\Delta} \\ \frac{V(x, y, z+\Delta) - V(x, y, z-\Delta)}{2\Delta} \end{bmatrix}
    $$
    If $\mathbf{x}$ lies on a boundary where neighbors are missing, use one-sided differences or clamp to the boundary value [G].
3.  **Normalize:** Compute the surface normal $\mathbf{n}$ by normalizing the gradient vector:
    $$
    \mathbf{n} = \frac{\nabla V(\mathbf{x})}{\|\nabla V(\mathbf{x})\|}
    $$
4.  **Shade:** Use $\mathbf{n}$ in a standard lighting model (e.g., Phong or Blinn-Phong) to compute the final color [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_shading(field, x, y, z, delta):
    """
    Computes the surface normal for gradient-based shading.
    field: 3D array (Nx, Ny, Nz) of voxel values.
    x, y, z: Continuous coordinates in grid space.
    delta: Grid spacing (assumed uniform).
    """
    def sample(vx, vy, vz):
        # Clamp to grid bounds and return value [G]
        ix = max(0, min(int(vx), field.shape[0] - 1))
        iy = max(0, min(int(vy), field.shape[1] - 1))
        iz = max(0, min(int(vz), field.shape[2] - 1))
        return field[ix, iy, iz]

    # Central differences sampling at x ± delta
    dx = (sample(x + delta, y, z) - sample(x - delta, y, z)) / (2 * delta)
    dy = (sample(x, y + delta, z) - sample(x, y - delta, z)) / (2 * delta)
    dz = (sample(x, y, z + delta) - sample(x, y, z - delta)) / (2 * delta)

    grad_norm = (dx*dx + dy*dy + dz*dz)**0.5

    if grad_norm < 1e-6:
        return [0.0, 0.0, 1.0] # Default normal for degenerate case [G]

    return [dx / grad_norm, dy / grad_norm, dz / grad_norm]
```

**Parameters**
*   **Delta ($\Delta$):** The physical or logical spacing between voxels. Typical value: 1.0 if using grid indices, or the physical resolution of the CT/MRI scan [G].
*   **Lighting Model:** Standard diffuse/specular coefficients are applied to the resulting normal [G].

**Edge Cases and Pitfalls**
*   **Homogeneous Regions:** In areas where the gradient magnitude is very low or zero (e.g., uniform tissue in CT data), the normal is undefined or noisy [S2, book]. The method is not robust here [S2, book].
*   **Noise Sensitivity:** Finite differences amplify high-frequency noise in the scalar field, leading to speckled shading artifacts [S1, unofficial].
*   **Multivariate Fields:** This method is unsuitable for multivariate fields [S2, book]. While the eigenvector $e_1$ (direction of greatest change) can be derived from the tensor $G$, it represents only orientation, not absolute direction [S2, book]. The sign of $e_1$ can flip in neighboring regions, causing interpolation issues and making it a poor choice for surface normals [S2, book].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment/pixel, but involves 6 additional texture fetches (or memory accesses) for central differences and a normalization step [G]. It is significantly slower than simple dot-product shading with pre-computed normals, being much as three times slower than evaluating dot products for a limited number of light sources in the fragment shader stage [S2, book].
*   **Memory Complexity:** $O(1)$ additional memory per sample, as no extra data structures are required beyond the scalar field itself [G].
*   **Strengths:** Provides intuitive shape cues for boundaries between materials by approximating normals from the scalar field gradient [S1, unofficial].
*   **Limitations:** Fails in homogeneous regions and is sensitive to noise [S2, book]. Not applicable to multivariate data [S2, book].
*   **When to use:** When visualizing boundaries in scalar volumes and explicit surface extraction is too expensive or unnecessary [G].
*   **When not to use:** For multivariate volumes, noisy data, or homogeneous regions where shadows or other techniques are preferred [S2, book].

## Background
Gradient-based surface shading was published by Hohne and Bernstein in 1986, utilizing the gray-level gradient to perform shading of surfaces rendered from 3D CT data [S3, unofficial]. This approach was part of early efforts in medical visualization to extract and visualize 3D hard and soft tissue surfaces [S3, unofficial]. It preceded the widespread adoption of Marching Cubes (1987) and volume raycasting (1988) [S3, unofficial].

## Sources
[S1] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
