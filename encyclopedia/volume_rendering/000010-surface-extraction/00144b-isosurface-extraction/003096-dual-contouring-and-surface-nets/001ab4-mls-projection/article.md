# MLS Projection

Also called: MLS projection via Hermite Point Set Surfaces, Moving Least Squares Surface, Parabolic Case Triangulation

## Overview
MLS Projection is an iterative method to project a query point onto a surface defined by a point set with normals, using Moving Least Squares (MLS) with Hermite data [S2, unofficial]. It builds on [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) by computing a local coordinate system and higher-order approximations via weighted accumulators, distinguishing it from standard MLS by explicitly tracking cross-product and dot-product terms for refined surface fitting [S1, unofficial]. Unlike [Surface Nets](921-surface-nets.md) which uses simple centroids, or [Dual Contouring](778-dual-contouring.md) which solves a linear system for a single vertex, MLS Projection refines a point's position and normal by minimizing the distance to the tangent planes of its neighbors, effectively smoothing noise and handling irregular point distributions [S3].

## How it works
**Inputs:**
*   `inputPoint`: The 3D query point to project ($P_0$).
*   `inputPositions`: Array of 3D points defining the surface.
*   `inputNormals`: Array of 3D normals corresponding to `inputPositions`.
*   `inputPointsetKdTree`: A spatial index (e.g., KD-Tree) for nearest neighbor search.
*   `knn`: Number of nearest neighbors to consider (default 20) [S2, unofficial].
*   `radius`: Radius parameter for the kernel function [S2, unofficial].
*   `kernel_type`: Integer selector for the kernel (0=Gaussian, 1=Wendland, 2=Singular) [S2, unofficial].
*   `numberOfIterations`: Number of iterations for the projection loop [S2, unofficial].

**Outputs:**
*   `outputPoint`: The projected 3D position on the surface.
*   `outputNormal`: The estimated normal at `outputPoint`.

**Data Structures:**
*   `Vec3`: A 3D vector structure with components `x, y, z` and operations: `dot(a, b)`, `normalize()`, `add(a, b)`, `sub(a, b)`, `scale(s)`.
*   `ANNidxArray`: Array of integer indices.
*   `ANNdistArray`: Array of floating-point squared distances.

**Step 1: Initialization**
Set `outputPoint` = `inputPoint` [S2, unofficial].

**Step 2: Iterative Projection**
Loop `iter` from 0 to `numberOfIterations - 1`:

1.  **Find Neighbors:**
    Query `inputPointsetKdTree` for the `knn` nearest neighbors to `outputPoint`.
    Retrieve indices into `id_nearest_neighbors` and squared distances into `square_distances_to_neighbors` [S2, unofficial].

2.  **Initialize Accumulators:**
    *   `sum_weight` = 0.0
    *   `sum_weighted_neighbors` = $(0, 0, 0)$
    *   `sum_weighted_normals` = $(0, 0, 0)$
    *   `sum_1` = 0.0
    *   `sum_2` = $(0, 0, 0)$
    *   `sum_3` = $(0, 0, 0)$
    *   `sum_4` = 0.0
    *   `u123` = $(0, 0, 0)$
    *   `u4` = 0.0
    *   `u0` = 0.0
    [S2, unofficial]

3.  **Accumulate Weights and Moments:**
    Loop `i` from 0 to `knn - 1`:
    *   `neighbor` = `inputPositions[id_nearest_neighbors[i]]`
    *   `neighborNormal` = `inputNormals[id_nearest_neighbors[i]]`
    *   `dist` = $\sqrt{\text{square\_distances\_to\_neighbors}[i]}$
    *   `weight` = `Get_kernel(kernel_type, dist, radius)`
    *   `sum_weight` += `weight`
    *   `sum_1` += `weight` * `dot(neighbor, neighborNormal)`
    *   `sum_2` += `weight` * `neighbor`
    *   `sum_3` += `weight` * `neighborNormal`
    *   `sum_4` += `weight` * `dot(neighbor, neighbor)`
    [S2, unofficial]

4.  **Compute Polynomial Coefficients:**
    *   $u_4 = 0.5 \times \frac{\text{sum\_1} - \frac{\text{dot}(\text{sum\_2}, \text{sum\_3})}{\text{sum\_weight}}}{\text{sum\_4} - \frac{\text{dot}(\text{sum\_2}, \text{sum\_2})}{\text{sum\_weight}}}$
    *   $u_{123} = \frac{\text{sum\_3} - 2 \times u_4 \times \text{sum\_2}}{\text{sum\_weight}}$
    *   $u_0 = -\frac{\text{dot}(\text{sum\_2}, u_{123}) + u_4 \times \text{sum\_4}}{\text{sum\_weight}}$
    [S2, unofficial]

5.  **Update Point and Normal:**
    *   Compute the implicit function value at `outputPoint`:
        $f = u_0 + \text{dot}(u_{123}, \text{outputPoint}) + u_4 \times \text{dot}(\text{outputPoint}, \text{outputPoint})$
    *   Compute the gradient (normal) at `outputPoint`:
        $\nabla f = u_{123} + 2 \times u_4 \times \text{outputPoint}$
    *   Normalize the gradient:
        $\mathbf{n} = \frac{\nabla f}{\|\nabla f\|}$
    *   Project `outputPoint` along the normal by the function value:
        $\text{diff} = f \times \mathbf{n}$
        $\text{outputPoint} = \text{outputPoint} - \text{diff}$
    *   Set `outputNormal` = $\mathbf{n}$
    [S2, unofficial]

    *Note: The source code also shows an alternative accumulation strategy for the first iteration or specific variants where `sum_weighted_neighbors` and `sum_weighted_normals` are used to compute a centroid and weighted normal average. However, the polynomial coefficient method (Steps 4-5) is the primary MLS implicit surface reconstruction mechanism described in the formulas [S2, unofficial]. The following pseudocode integrates the polynomial approach as the core definition.*

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mls_projection(inputPoint, inputPositions, inputNormals, kdtree, knn, radius, kernel_type, numberOfIterations):
    outputPoint = inputPoint
    outputNormal = Vec3(0, 0, 0)

    for iter in range(numberOfIterations):
        # 1. Find Neighbors
        id_nearest_neighbors, square_distances_to_neighbors = kdtree.knearest(outputPoint, knn)
        
        # 2. Initialize Accumulators
        sum_weight = 0.0
        sum_1 = 0.0
        sum_2 = Vec3(0, 0, 0)
        sum_3 = Vec3(0, 0, 0)
        sum_4 = 0.0
        
        # 3. Accumulate
        for i in range(knn):
            neighbor = inputPositions[id_nearest_neighbors[i]]
            neighborNormal = inputNormals[id_nearest_neighbors[i]]
            dist = sqrt(square_distances_to_neighbors[i])
            weight = get_kernel(kernel_type, dist, radius)
            
            sum_weight += weight
            sum_1 += weight * dot(neighbor, neighborNormal)
            sum_2 += weight * neighbor
            sum_3 += weight * neighborNormal
            sum_4 += weight * dot(neighbor, neighbor)
            
        # 4. Compute Coefficients
        # Avoid division by zero if sum_weight is 0 (should not happen with valid knn)
        if sum_weight == 0:
            break
            
        denom_4 = sum_4 - dot(sum_2, sum_2) / sum_weight
        if abs(denom_4) < 1e-9:
            # Degenerate case: neighbors are collinear or coincident
            # Fallback to simple weighted average of normals and positions
            sum_weighted_neighbors = Vec3(0, 0, 0)
            sum_weighted_normals = Vec3(0, 0, 0)
            for i in range(knn):
                neighbor = inputPositions[id_nearest_neighbors[i]]
                neighborNormal = inputNormals[id_nearest_neighbors[i]]
                dist = sqrt(square_distances_to_neighbors[i])
                weight = get_kernel(kernel_type, dist, radius)
                sum_weighted_neighbors += weight * neighbor
                sum_weighted_normals += weight * neighborNormal
            
            outputPoint = sum_weighted_neighbors / sum_weight
            outputNormal = sum_weighted_normals.normalize()
            break

        u4 = 0.5 * ((sum_1 - dot(sum_2, sum_3) / sum_weight) / denom_4)
        u123 = (sum_3 - 2 * u4 * sum_2) / sum_weight
        u0 = -(dot(sum_2, u123) + u4 * sum_4) / sum_weight
        
        # 5. Update Point and Normal
        # Implicit function value f(p) = u0 + u123 . p + u4 * |p|^2
        f_val = u0 + dot(u123, outputPoint) + u4 * dot(outputPoint, outputPoint)
        
        # Gradient n(p) = u123 + 2 * u4 * p
        grad = u123 + 2 * u4 * outputPoint
        outputNormal = grad.normalize()
        
        # Project along normal
        diff = f_val * outputNormal
        outputPoint = outputPoint - diff
        
    return outputPoint, outputNormal

def get_kernel(kernel_type, r, h):
    if kernel_type == 0: # Gaussian
        return exp(-(r * r) / (h * h))
    elif kernel_type == 1: # Wendland
        if r < h:
            return (1 - r / h)**4 * (1 + 4 * r / h)
        return 0.0
    elif kernel_type == 2: # Singular
        return (r / h)**(-2) # w=2 is common default if not specified, source says (r/h)^-w
    return 0.0
```

**Edge Cases and Pitfalls:**
*   **Degenerate Neighbors:** If `knn` neighbors are collinear or coincident, the denominator in $u_4$ calculation becomes zero. A fallback to simple weighted averaging is required [G].
*   **Parabolic Case:** If the local surface is parabolic ($\lambda_1 > 0, \lambda_2 = 0$), the standard quadratic fit may be ill-conditioned. The error profile is determined by the projection onto the first eigenvector $e_1$, and the cluster width is limited to $\sqrt{4\epsilon/\lambda_1}$ [S1, unofficial].
*   **Precision:** The iterative projection can suffer from floating-point precision errors if the surface is very flat or if the radius is too small relative to the point density [S3].

## Subtopics

### [APSS (Approximate Projected Point Set Surfaces)](843-apss-approximate-projected-point-set-surfaces.md)
APSS is an iterative method to project a query point onto a surface defined by a point set with normals, using Moving Least Squares (MLS). It builds on [MLS Projection](823-mls-projection.md) by computing a local coordinate system and higher-order approximations via weighted accumulators, distinguishing it from standard MLS by explicitly tracking cross-product and dot-product terms for refined surface fitting. Choose APSS when higher-order surface continuity is required and the computational cost of additional accumulators is acceptable. Link: [APSS](843-apss-approximate-projected-point-set-surfaces.md).

### [Moving Least Squares Implicit Surface Reconstruction](5328-moving-least-squares-implicit-surface-reconstruction.md)
Moving Least Squares (MLS) Implicit Surface Reconstruction is a technique for reconstructing a surface from an unordered 3D point cloud by creating an implicit function field using Moving Least Squares, from which isosurfaces are extracted. It differs from its parent [MLS Projection](823-mls-projection.md) by constructing a global implicit field defined by local quadratic fits rather than iteratively projecting query points, and from its sibling [APSS](843-apss-approximate-projected-point-set-surfaces.md) by relying on a standard MLS approximation of the signed distance function rather than approximate projected point set surfaces. Choose this method when a global implicit field is needed for further processing (e.g., boolean operations) rather than just a mesh. Link: [Moving Least Squares Implicit Surface Reconstruction](5328-moving-least-squares-implicit-surface-reconstruction.md).

### Kernel Functions for MLS
A set of kernel functions used to weight neighbors in MLS surface reconstruction based on distance $r$ and scale $h$ [S2, unofficial].
*   **Gaussian kernel** (`kernel_type=0`): $K(r) = e^{-r^2/h^2}$ [S2, unofficial].
*   **Wendland kernel** (`kernel_type=1`): $K(r) = (1 - r/h)^4 (1 + 4r/h)$ [S2, unofficial].
*   **Singular kernel** (`kernel_type=2`): $K(r) = (r/h)^{-w}$ [S2, unofficial].
Choose Gaussian for smoothness, Wendland for compact support (faster computation), and Singular for specific weighting behaviors.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K \cdot I)$, where $N$ is the number of query points, $K$ is `knn`, and $I$ is `numberOfIterations`. KD-Tree search is $O(\log N)$ per neighbor, so total is $O(N \cdot K \log N \cdot I)$ if building the tree is included, or $O(N \cdot K \cdot I)$ if the tree is pre-built [S3].
*   **Memory Complexity:** $O(N)$ to store the point cloud and KD-Tree [S3].
*   **Measured Performance:** Implemented on a PC with dual-core AMD Opteron 2.25GHz, 4GB RAM. Multi-threaded implementation allows parallel processing of independent surface parts [S3].
*   **Strengths:** Handles noisy data well due to smoothing effect; works with irregular point distributions; produces smooth surfaces [S3].
*   **Limitations:** Computationally expensive due to iterative nature and least-squares solving; sensitive to parameter choice (`knn`, `radius`); may struggle with sharp features if not carefully tuned [S3].
*   **When to use:** When input data is noisy or irregular, and smooth surface reconstruction is prioritized over speed.
*   **When not to use:** For real-time applications with strict latency constraints, or when the input data is already clean and regularly sampled (use Marching Cubes or Surface Nets instead).

## Background
MLS Projection was developed as part of the Hermite Point Set Surfaces (HPSS) method. The technique was introduced by Tamy Boubekeur and others in the context of multiresolution modeling and scientific visualization [S1, unofficial]. The specific implementation details, including kernel functions and iterative projection, are documented in the GLRK library and associated research papers [S2, unofficial]. The underlying MLS theory for implicit surface reconstruction was further explored by others, such as in "High-Quality Extraction of Isosurfaces from Regular and Irregular Grids" [S3]. The method is used in computer graphics for surface reconstruction from point clouds and in scientific visualization for smoothing and approximating complex data sets [S1, unofficial][S3].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S3] High-Quality Extraction of Isosurfaces from Regular and Irregular Grids, https://doi.org/10.1109/tvcg.2006.149, fetched 2026-10-26
