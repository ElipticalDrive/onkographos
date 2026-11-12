# Implicit Surface Field Function Evaluation

Also called: Field Function Evaluation, Implicit Function Evaluation

## Overview
Implicit Surface Field Function Evaluation computes the scalar value $d = f(\mathbf{x})$ for an implicit surface at query points $\mathbf{x}$, where $d$ represents the signed distance or density. It solves the problem of determining whether a point lies inside, outside, or on the contour of a volume, which is fundamental for ray marching and isosurface extraction. It differs from [Batched SDF Inference](2329-batched-sdf-inference.md) by operating on arbitrary single points rather than fixed neural network batches, and from [DeepSDF Neural Network](3051-deepsdf-neural-network.md) by using explicit analytical or grid-based storage rather than learned latent codes.

## How it works
**Inputs and Outputs:**
- Input: Query point coordinates $(x_0, y_0, z_0)$, step size $dz$, number of points $num\_points$.
- Output: Array of scalar density/distance values $densities$.

**Data Structures:**
- `Isosurface`: Interface class containing the field function implementation.
- `densities`: Array of floats of size $num\_points$.

**Steps:**
1. If the implicit surface has an associated transformation matrix $T$, compute the inverse transformation $T^{-1}$.
2. Transform the query point $\mathbf{x}_{world} = (x_0, y_0, z_0)$ by $T^{-1}$ to obtain the local coordinate $\mathbf{x}_{local} = (x_t, y_t, z_t)$. If no transformation is applied, $T^{-1}$ is the identity, so $\mathbf{x}_{local} = \mathbf{x}_{world}$.
3. Evaluate the primitive field function $f(\mathbf{x}_{local})$ to get the signed distance $d$.
4. Store $d$ in the output array.
5. Increment the query point along the +z axis: $z_0 \leftarrow z_0 + dz$.
6. Repeat steps 2-5 for $num\_points$.

**Formulas:**
For a sphere of radius $R$ centered at the origin:
$$ f(x,y,z) = x^2 + y^2 + z^2 - R^2 $$
- $d < 0$: Point is inside the volume.
- $d > 0$: Point is outside the volume.
- $d = 0$: Point is on the contour.

**Parameters:**
- $dz$: Delta step size along the +z axis for batch evaluation.
- $num\_points$: Number of points to evaluate in the batch.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_field_function(x0, y0, z0, dz, num_points, radius, inverse_transform_matrix):
    """
    Evaluates the implicit field function for a sphere along the +z axis.
    
    Args:
        x0, y0, z0: Starting coordinates of the query point.
        dz: Step size along the z-axis.
        num_points: Number of points to evaluate.
        radius: Radius of the sphere.
        inverse_transform_matrix: 4x4 matrix representing T^-1.
        
    Returns:
        densities: List of float values representing the field function output.
    """
    densities = []
    sqr_rad = radius * radius
    
    for i in range(num_points):
        # Step 1: Transform query point to local space
        # Assuming inverse_transform_matrix is a 4x4 matrix
        # Homogeneous coordinate transformation
        x_local = inverse_transform_matrix[0][0] * x0 + inverse_transform_matrix[0][1] * y0 + inverse_transform_matrix[0][2] * z0 + inverse_transform_matrix[0][3]
        y_local = inverse_transform_matrix[1][0] * x0 + inverse_transform_matrix[1][1] * y0 + inverse_transform_matrix[1][2] * z0 + inverse_transform_matrix[1][3]
        z_local = inverse_transform_matrix[2][0] * x0 + inverse_transform_matrix[2][1] * y0 + inverse_transform_matrix[2][2] * z0 + inverse_transform_matrix[2][3]
        
        # Step 2: Evaluate sphere field function
        sqr_dist = x_local * x_local + y_local * y_local + z_local * z_local
        d = sqr_dist - sqr_rad
        
        # Step 3: Store result
        densities.append(d)
        
        # Step 4: Increment z for next point
        z0 = z0 + dz
        
    return densities
```

**Edge Cases and Pitfalls:**
- Precision: Floating-point errors may cause $d$ to be slightly non-zero for points exactly on the contour.
- Boundaries: The bounding box of the implicit surface must be computed and reported to optimize culling, as it is not always trivial to derive from the field function alone [S1, unofficial].

## Subtopics

### [SDF Evaluation Implementation](1658-sdf-evaluation-implementation.md)
Implements runtime evaluation of an SDF defined on a 3D voxel grid, supporting configurable interpolation orders. Choose this when using grid-based storage rather than analytical primitives.

## Operating characteristics
- Time Complexity: $O(num\_points)$ per evaluation, assuming constant-time primitive evaluation.
- Memory Complexity: $O(1)$ auxiliary space for analytical primitives; $O(V)$ for grid-based implementations where $V$ is the number of voxels.
- Strengths: Compact and precise representation of volumes; efficient for simple primitives.
- Limitations: Difficult to determine bounding boxes for complex implicit functions; limited to specific geometric shapes or grid-based approximations.
- When to use: Simple analytical shapes or pre-computed voxel grids.
- When not to use: Complex, non-analytical surfaces requiring neural network inference or high-resolution grid storage.

## Background
Implicit surfaces were introduced as a compact and precise way to describe volumes in 3D space, avoiding the need for explicit polygonal approximations. The concept is foundational for contouring algorithms like Marching Cubes and Dual Contouring. The specific implementation details described here, including the `Isosurface` interface and batch evaluation along the +z axis, are documented in an unofficial source by Zukkers [S1, unofficial].

## Sources
[S1] Implicit Surfaces, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
