# MLS Projection

Also called: MLS projection via Hermite Point Set Surfaces, Moving Least Squares Surface, Parabolic Case Triangulation

## Overview
MLS Projection is an iterative algorithm that projects a query point onto a surface defined by a set of input points and their associated normals using Moving Least Squares (MLS) approximation [S2, unofficial]. It solves the problem of finding the closest point on a noisy or implicit surface defined by Hermite data (positions and normals) by fitting a local quadratic function and minimizing the distance to the tangent planes of neighboring points [S2, unofficial]. Unlike standard Marching Cubes which relies on linear interpolation within grid cells, MLS Projection constructs a smooth, locally fitted surface geometry, making it suitable for surface reconstruction from point clouds or for refining isosurfaces extracted from scalar fields [S3]. It distinguishes itself from its parent [Isosurface extraction](30-isosurface-extraction.md) by operating on unstructured point sets with normals rather than structured grid values, and from its sibling [APSS](843-apss-approximate-projected-point-set-surfaces.md) by using a direct iterative projection on tangent planes rather than higher-order approximate projected point set surfaces [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input**: A set of input points $P_{in}$ with associated normals $N_{in}$, a query point $q$, a number of nearest neighbors $knn$, a search radius $radius$, a kernel type $kernel\_type$, and a maximum number of iterations $numberOfIterations$.
*   **Output**: A projected point $p_{out}$ on the surface and its corresponding normal $n_{out}$.

### Data Structures
*   **Point Set**: Arrays or vectors storing 3D positions (`Vec3`) and 3D normals (`Vec3`).
*   **K-D Tree**: A spatial index structure (e.g., `inputPointsetKdTree`) built on the input positions to efficiently retrieve the $knn$ nearest neighbors [S2, unofficial].
*   **Accumulators**: Floating-point scalars and 3D vectors used to store weighted sums during the neighbor loop.

### Parameters
*   $knn$: Number of nearest neighbors. Default: 20 [S2, unofficial].
*   $radius$: Search radius for the kernel function [S2, unofficial].
*   $numberOfIterations$: Number of iterations for the projection loop [S2, unofficial].
*   $kernel\_type$: Selector for the weighting function (0=Gaussian, 1=Wendland, 2=Singular) [S2, unofficial].

### Algorithm Steps

1.  **Initialization**: Set the current point $p_{curr} = q$ and the current normal $n_{curr}$ (often initialized to the normal at $q$ if available, or zero if not; in the standard HPSS projection loop, the normal is updated iteratively based on neighbors) [S2, unofficial]. *Note: The provided source code initializes `outputPoint` to `inputPoint` but does not explicitly initialize `outputNormal` before the loop in the snippet, implying it may be derived from the first iteration or passed in. We assume $n_{curr}$ is initialized to the normal of the query point if available, or $(0,0,0)$ if not, but the update step normalizes the weighted sum, so initialization of $n_{curr}$ is less critical than the accumulation.*

2.  **Iteration Loop**: For $iter$ from 0 to $numberOfIterations - 1$:
    a.  **Neighbor Search**: Find the $knn$ nearest neighbors of $p_{curr}$ in the input point set using the K-D tree. Retrieve their indices $id_{neighbors}$ and squared distances $d^2_{i}$ [S2, unofficial].
    b.  **Accumulator Initialization**: Initialize accumulators:
        *   $sum\_weight = 0.0$
        *   $sum\_weighted\_neighbors = (0, 0, 0)$
        *   $sum\_weighted\_normals = (0, 0, 0)$
    c.  **Neighbor Processing**: For each neighbor $i$ from 0 to $knn-1$:
        i.  Get neighbor position $p_i$ and normal $n_i$ from the input data using $id_{neighbors}[i]$.
        ii. **Projection onto Tangent Plane**: Compute the projection of the current point $p_{curr}$ onto the tangent plane of the neighbor $i$. The tangent plane is defined by $n_i$ passing through $p_i$.
            $$ ProjOnTang_i = p_{curr} - ((p_{curr} - p_i) \cdot n_i) n_i $$
        iii. **Weight Calculation**: Compute the weight $w_i$ using the kernel function $K$ based on the distance $r_i = \sqrt{d^2_i}$ and radius $h = radius$:
            $$ w_i = K(r_i, h) $$
            (See *Kernel Functions for MLS* below for specific formulas).
        iv. **Update Accumulators**:
            $$ sum\_weighted\_neighbors \leftarrow sum\_weighted\_neighbors + w_i \cdot ProjOnTang_i $$
            $$ sum\_weight \leftarrow sum\_weight + w_i $$
            $$ sum\_weighted\_normals \leftarrow sum\_weighted\_normals + w_i \cdot n_i $$
    d.  **Update Point and Normal**:
        i.  Compute the new center point $c$:
            $$ c = \frac{sum\_weighted\_neighbors}{sum\_weight} $$
        ii. Normalize the weighted normal sum to get the new normal $n$:
            $$ n = \frac{sum\_weighted\_normals}{\| sum\_weighted\_normals \|} $$
        iii. Update the current point and normal for the next iteration:
            $$ p_{curr} = c $$
            $$ n_{curr} = n $$
    e.  **Memory Cleanup**: Free temporary arrays for neighbor indices and distances [S2, unofficial].

3.  **Final Output**: After the loop, set $p_{out} = p_{curr}$ and $n_{out} = n_{curr}$ [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mls_projection(input_positions, input_normals, kdtree, query_point, knn=20, radius=1.0, kernel_type=0, num_iterations=10):
    """
    Projects a query point onto the surface defined by input_positions and input_normals
    using Moving Least Squares (MLS) via Hermite Point Set Surfaces (HPSS).
    """
    current_point = query_point
    current_normal = (0.0, 0.0, 0.0) # Initialize normal; often derived from first iter or passed in
    
    for iter in range(num_iterations):
        # 1. Find k-nearest neighbors
        neighbor_indices, squared_distances = kdtree.knearest(current_point, knn)
        
        sum_weight = 0.0
        sum_weighted_neighbors = (0.0, 0.0, 0.0)
        sum_weighted_normals = (0.0, 0.0, 0.0)
        
        for i in range(knn):
            idx = neighbor_indices[i]
            dist_sq = squared_distances[i]
            dist = sqrt(dist_sq)
            
            neighbor_pos = input_positions[idx]
            neighbor_norm = input_normals[idx]
            
            # 2. Project current point onto neighbor's tangent plane
            # Proj = p_curr - dot(p_curr - p_i, n_i) * n_i
            diff_vec = vector_sub(current_point, neighbor_pos)
            dot_val = vector_dot(diff_vec, neighbor_norm)
            proj_on_tangent = vector_sub(current_point, vector_scale(neighbor_norm, dot_val))
            
            # 3. Compute weight
            weight = get_kernel(kernel_type, dist, radius)
            
            # 4. Update accumulators
            sum_weighted_neighbors = vector_add(sum_weighted_neighbors, vector_scale(proj_on_tangent, weight))
            sum_weight += weight
            sum_weighted_normals = vector_add(sum_weighted_normals, vector_scale(neighbor_norm, weight))
            
        # 5. Compute new point and normal
        if sum_weight == 0:
            # Fallback if no neighbors found (e.g., out of bounds)
            break
            
        new_center = vector_scale(sum_weighted_neighbors, 1.0 / sum_weight)
        
        # Normalize the weighted normal
        norm_len = vector_length(sum_weighted_normals)
        if norm_len > 1e-8:
            new_normal = vector_scale(sum_weighted_normals, 1.0 / norm_len)
        else:
            new_normal = (0.0, 0.0, 1.0) # Arbitrary default
            
        current_point = new_center
        current_normal = new_normal
        
    return current_point, current_normal

def get_kernel(kernel_type, r, h):
    if kernel_type == 0: # Gaussian
        return exp(-(r * r) / (h * h))
    elif kernel_type == 1: # Wendland
        if r < h:
            return (1 - r/h)**4 * (1 + 4*r/h)
        return 0.0
    elif kernel_type == 2: # Singular
        return (r/h)**(-2) # w=2 is common, source says (r/h)^-w
    return 0.0
```

### Edge Cases and Pitfalls
*   **Zero Weight**: If `sum_weight` is zero (e.g., query point is far from all input points), the division in step 5 fails. A fallback or error handling is required [G].
*   **Degenerate Normals**: If the weighted normal sum has zero length, normalization fails. This can happen if neighbors have conflicting normals that cancel out. A default normal or the query point's normal should be used [G].
*   **Parabolic Case**: In the context of surface triangulation from MLS fields, if the quadratic form's eigenvalues result in one zero eigenvalue ($\lambda_1 > 0, \lambda_2 = 0$), the surface is parabolic. The triangulation must handle this by transforming the canonical form $p(\bar{x}, \bar{y}) = \bar{x}^2$ using the eigenvector $e_1$ and placing vertices at a specific distance $\epsilon$ [S1, unofficial]. This affects the *mesh generation* phase, not the point projection itself, but is a critical pitfall if MLS is used to define the implicit field for meshing.
*   **Precision**: The iterative nature can suffer from floating-point precision issues if the number of iterations is too high or the radius is too small relative to the point density [S3].

## Subtopics

### Kernel Functions for MLS
Kernel functions determine the weight of each neighbor based on its distance to the query point. They are crucial for the smoothness and locality of the MLS fit [S2, unofficial].
*   **Gaussian** (`kernel_type=0`): $K(r) = e^{-r^2/h^2}$. Provides smooth, rapidly decaying weights [S2, unofficial].
*   **Wendland** (`kernel_type=1`): $K(r) = (1 - r/h)^4 (1 + 4r/h)$ for $r < h$, else 0. Compact support, ensuring only neighbors within radius $h$ contribute [S2, unofficial].
*   **Singular** (`kernel_type=2`): $K(r) = (r/h)^{-w}$. Can be used for different weighting behaviors, potentially emphasizing closer points more strongly [S2, unofficial].

## Operating characteristics

*   **Time Complexity**: $O(N \cdot k \cdot I)$ per query point, where $N$ is the number of input points, $k$ is $knn$, and $I$ is the number of iterations. With a K-D tree, neighbor search is $O(\log N)$, making it $O(k \cdot I \cdot \log N)$ per query. Total time for $M$ query points is $O(M \cdot k \cdot I \cdot \log N)$.
*   **Memory Complexity**: $O(N)$ to store the input point set and K-D tree.
*   **Performance**: The method is computationally expensive due to the iterative nature and the need to solve a local least-squares problem (implicitly via weighted averages) at each step [S3]. It is significantly slower than linear interpolation methods like Marching Cubes for dense grids but produces smoother results for sparse, noisy point clouds [S3].
*   **Strengths**: Handles noisy data well; produces smooth surfaces; works with unstructured data (point clouds); does not require a regular grid [S3].
*   **Limitations**: Slow for large datasets; sensitive to parameter choice ($knn$, $radius$); requires normals as input; can produce artifacts if the point distribution is highly non-uniform [S3].
*   **When to Use**: Surface reconstruction from scanned data (point clouds); smoothing noisy implicit fields; applications where smoothness is prioritized over speed.
*   **When Not to Use**: Real-time rendering of dense grid-based data (use Marching Cubes); data without normals (requires estimation first).

## Background

MLS Projection, specifically in the context of Hermite Point Set Surfaces (HPSS), was developed as part of the broader Moving Least Squares method for surface reconstruction. The method was popularized in the mid-2000s for high-quality extraction of isosurfaces from irregular grids and point clouds. The specific iterative projection algorithm described here is implemented in the GLRK (OpenGL Refinement Kernel) library by Tamy Boubekeur [S2, unofficial]. The theoretical foundation relies on fitting local polynomial approximations to weighted data points, a technique originally introduced by Levin (2003) and adapted for implicit surfaces by others [S3]. The handling of the parabolic case in triangulation is detailed in multiresolution modeling literature [S1, unofficial].

## Sources

[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S3] High-Quality Extraction of Isosurfaces from Regular and Irregular Grids, https://doi.org/10.1109/tvcg.2006.149, fetched 2026-10-26
