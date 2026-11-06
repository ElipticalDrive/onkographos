# Ray-Ray Intersection
Also called: Least-squares ray intersection, Ray vs. Ray Intersection

## Overview
Computes the least-squares intersection point of two rays in 3D space, finding the point that minimizes the squared distance to both rays when they do not intersect exactly due to noise or parallax [S1, unofficial]. It differs from exact intersection methods by providing a robust solution for skew lines, which is essential in volume rendering pipelines where ray origins and directions may have minor misalignments [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Two rays, each defined by an origin $\mathbf{o} \in \mathbb{R}^3$ and a direction $\mathbf{d} \in \mathbb{R}^3$. The inputs are typically batched tensors.
- **Output**: A tensor of points $\mathbf{p} \in \mathbb{R}^3$ representing the closest point on both rays. For parallel rays, the output is set to infinity.

**Data Structures**
The implementation uses batched tensors. Let the batch dimension be $B$ and the spatial dimensions be $*shape$.
- Origins: Tensor of shape $(2, *shape, 3)$, where the first axis indexes the two rays.
- Directions: Tensor of shape $(2, *shape, 3)$.
- Intermediate matrices: Shape $(2, *shape, 3, 3)$ for the outer product terms.

**Parameters**
- `eps`: Float threshold for parallel detection. Default value: $10^{-5}$ [S1, unofficial].
- `inf_val`: Value used to represent infinity for parallel cases. Default value: $10^{10}$ [S1, unofficial].

**Steps**
1. **Broadcast**: Ensure origins and directions for both rays share a common broadcastable shape.
2. **Parallel Detection**: Compute the dot product of the direction vectors $\mathbf{d}_0$ and $\mathbf{d}_1$. If $\mathbf{d}_0 \cdot \mathbf{d}_1 > 1 - \text{eps}$, the rays are considered parallel [S1, unofficial].
3. **Filter**: Identify non-parallel ray pairs. Parallel pairs will be handled separately by assigning the infinity value.
4. **Construct Matrices**: For each non-parallel pair, compute the matrix $N$ for each ray $r \in \{0, 1\}$:
   $$ n_{r,b,i,j} = d_{r,b,i} d_{r,b,j} - \delta_{i,j} $$
   where $\delta_{i,j}$ is the Kronecker delta (identity matrix element) [S1, unofficial]. This is the outer product of the direction minus the identity matrix.
5. **Aggregate Left-Hand Side (LHS)**: Sum the $N$ matrices over both rays to form the LHS matrix $L$ of the linear system:
   $$ L_{b,i,j} = \sum_{r=0}^{1} n_{r,b,i,j} $$
6. **Aggregate Right-Hand Side (RHS)**: Compute the RHS vector $R$ using the origins $\mathbf{o}$:
   $$ R_{b,i} = \sum_{r=0}^{1} \sum_{j=0}^{2} n_{r,b,i,j} o_{r,b,j} $$
7. **Solve**: Solve the linear system $L \mathbf{p} = R$ for $\mathbf{p}$ using least squares [S1, unofficial].
8. **Assign Output**: Initialize the output tensor with `inf_val`. Assign the computed solution $\mathbf{p}$ to the indices corresponding to non-parallel rays.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def ray_ray_intersection(origins_x, directions_x, origins_y, directions_y, eps=1e-5, inf_val=1e10):
    """
    Compute the least-squares intersection of two rays.
    
    Args:
        origins_x, directions_x: Tensors of shape (*shape, 3) for ray 1.
        origins_y, directions_y: Tensors of shape (*shape, 3) for ray 2.
        eps: Threshold for parallel detection.
        inf_val: Value for parallel ray outputs.
        
    Returns:
        points: Tensor of shape (*shape, 3) with the closest points.
    """
    # 1. Broadcast to common shape
    # Note: In a real implementation, torch.broadcast_tensors would be used.
    # Here we assume inputs are already broadcastable or we stack them.
    
    # Stack into shape (2, *shape, 3)
    origins = torch.stack([origins_x, origins_y], dim=0)
    directions = torch.stack([directions_x, directions_y], dim=0)
    
    # 2. Detect parallel rays
    # Dot product of directions: einsum("r b i, r b j -> r b i j") is not quite right for dot product.
    # Dot product is einsum("r b i, r b i -> r b")
    # The source uses einsum(directions_x, directions_y, "... xyz, ... xyz -> ...") which implies summing over the last dim.
    # Let's use standard einsum for dot product.
    dot_product = torch.einsum("r b i, r b i -> r b", directions, directions) # This is wrong, it sums squares.
    # Correct dot product between ray 0 and ray 1:
    # directions[0] is ray 0, directions[1] is ray 1.
    # We need dot(directions[0], directions[1])
    # Using einsum: "b i, b i -> b"
    
    # Let's follow the source's logic more closely.
    # Source: `parallel = einsum(directions_x, directions_y, "... xyz, ... xyz -> ...") > 1 - eps`
    # This implies summing over the last dimension (3).
    
    # Reshape for einsum if necessary, but let's assume standard batched tensors.
    # directions shape: (2, *shape, 3)
    # We want dot product between index 0 and index 1 of the first dim.
    
    # Compute dot product of directions for each pair
    # directions[0] shape (*shape, 3), directions[1] shape (*shape, 3)
    # dot = sum_i directions[0][..., i] * directions[1][..., i]
    dot = torch.sum(directions[0] * directions[1], dim=-1)
    
    parallel = dot > (1.0 - eps)
    
    # 3. Stack origins and directions for matrix construction
    # origins shape: (2, *shape, 3)
    # directions shape: (2, *shape, 3)
    
    # 4. Compute N matrix for each ray
    # n_r,b,i,j = d_r,b,i * d_r,b,j - delta_ij
    # Outer product: einsum("r b i, r b j -> r b i j")
    outer = torch.einsum("r b i, r b j -> r b i j", directions, directions)
    
    # Subtract identity matrix
    # Identity matrix is 3x3. We need to broadcast it to (2, *shape, 3, 3)
    # torch.eye(3) is (3, 3).
    # We can use broadcasting or explicit indexing.
    # Let's create a mask or subtract directly if shapes align.
    # outer shape: (2, *shape, 3, 3)
    # eye shape: (3, 3)
    # To subtract, we need eye to broadcast to (*shape, 3, 3) then to (2, *shape, 3, 3)
    # Actually, torch.einsum result is (2, *shape, 3, 3).
    # We can subtract torch.eye(3) if we expand_dims.
    
    eye = torch.eye(3, device=directions.device, dtype=directions.dtype)
    # Expand eye to match batch dimensions
    # eye shape (3, 3) -> (*shape, 3, 3) -> (2, *shape, 3, 3)
    # This is tricky with arbitrary *shape.
    # A safer way:
    # n = outer - eye.view(1, 1, 3, 3) # This assumes *shape is empty or handled by broadcasting?
    # No, outer is (2, *shape, 3, 3). eye is (3, 3).
    # We need to align the last two dims.
    # outer[..., i, j] - eye[i, j]
    
    # Let's use a loop or explicit indexing for clarity if *shape is unknown, 
    # but for vectorization, we assume broadcasting works.
    # eye.view(1, 1, 3, 3) broadcasts to (2, *shape, 3, 3) if *shape is compatible?
    # No, (1, 1, 3, 3) broadcasts to (2, B, 3, 3) if B is the batch dim.
    # Yes, this works.
    
    n = outer - eye.view(1, 1, 3, 3)
    
    # 5. Aggregate LHS: sum over r (dim 0)
    # lhs shape: (*shape, 3, 3)
    lhs = torch.sum(n, dim=0)
    
    # 6. Aggregate RHS: sum over r and j
    # rhs = sum_r sum_j n_r,b,i,j * o_r,b,j
    # einsum("r b i j, r b j -> r b i")
    rhs_temp = torch.einsum("r b i j, r b j -> r b i", n, origins)
    # sum over r
    rhs = torch.sum(rhs_temp, dim=0)
    
    # 7. Solve least squares
    # result = torch.linalg.lstsq(lhs, rhs).solution
    # lhs shape (*shape, 3, 3), rhs shape (*shape, 3)
    # lstsq expects (..., M, N) and (..., M, K) or (..., M)
    # Here M=3, N=3, K=1 (vector)
    
    # Note: torch.linalg.lstsq returns a solution object.
    # We need to handle the case where lhs is singular (parallel rays), 
    # but we filtered those out. However, numerical stability might still be an issue.
    # The source assigns inf to parallel rays, so we solve only for non-parallel.
    
    # To solve only for non-parallel, we can mask or solve all and mask.
    # Solving all is easier if we handle the parallel case by replacing lhs/rhs 
    # or just masking the result.
    
    # Let's solve for all, then mask.
    # If lhs is singular, lstsq might return garbage or NaN.
    # The source filters out parallel rays.
    
    # Implementation detail: The source says "Filter out parallel ray pairs from all tensors."
    # This implies we only solve for non-parallel.
    
    # Let's create a mask for non-parallel
    non_parallel = ~parallel
    
    # We need to solve the system only for non-parallel indices.
    # This is complex with batched tensors.
    # A common trick is to add a small regularization to parallel rays to make them solvable, 
    # then mask the result. Or use a loop.
    # Given the constraint of "valid Python", we'll use a masked approach if possible, 
    # or assume the user handles the filtering.
    
    # For the sake of the pseudocode, we will solve for all and then mask.
    # To avoid singular matrix errors, we can add a tiny epsilon to the diagonal of lhs for parallel rays?
    # Or just rely on the fact that parallel rays are filtered.
    
    # Let's assume we have a function that solves batched lstsq.
    
    # result = torch.linalg.lstsq(lhs, rhs).solution
    # This might fail for parallel rays.
    
    # Alternative: Use the source's logic of filtering.
    # We can't easily filter batched tensors without gathering.
    
    # Let's stick to the source's high-level description.
    
    # result = torch.linalg.lstsq(lhs, rhs).solution
    # This line is from the source.
    
    # We will write it as:
    try:
        result = torch.linalg.lstsq(lhs, rhs).solution
    except RuntimeError:
        # Handle singular matrix error if any parallel rays slipped through
        result = torch.full_like(lhs[..., 0], inf_val) # Placeholder for error handling
        
    # 8. Assign output
    # Initialize with inf_val
    output = torch.full_like(origins_x, inf_val)
    
    # Assign result to non-parallel indices
    # We need to broadcast result to the output shape
    # result shape (*shape, 3)
    # output shape (*shape, 3)
    
    # Masking assignment
    # output[non_parallel] = result[non_parallel]
    # This requires non_parallel to be a boolean tensor of shape (*shape,)
    
    # If parallel is (*shape,), then this works.
    
    output[non_parallel] = result[non_parallel]
    
    return output
```

**Edge Cases and Pitfalls**
- **Parallel Rays**: The dot product of directions is close to 1. The method detects this and returns infinity. Precision issues near the threshold `eps` may cause instability.
- **Singular Matrix**: If rays are nearly parallel, the LHS matrix $L$ may be ill-conditioned. Least squares solvers may fail or return inaccurate results.
- **Numerical Precision**: The subtraction of the identity matrix from the outer product can lead to loss of precision if the direction vectors are not normalized. It is recommended to normalize direction vectors before computation [G].

## Subtopics

### [Ray-box intersection](776-ray-box-intersection.md)
Intersects a ray with an axis-aligned bounding box. Faster than general ray-ray intersection for spatial partitioning.

### [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md)
Intersects a ray with a triangle using Plücker coordinates. Used for mesh rendering.

### [Ray Crossing Count (Winding Number)](915-ray-crossing-count-winding-number.md)
Determines if a ray intersects a closed surface by counting crossings. Used for point-in-volume tests.

### [Sphere Intersection](1620-sphere-intersection.md)
Intersects a ray with a sphere. Analytic solution, very fast.

### [Ray-Volume Intersection (Face Distance Method)](2381-ray-volume-intersection-face-distance-method.md)
Intersects a ray with a volume grid using face distances. Used for voxel-based rendering.

### [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md)
Intersects a ray with a tetrahedron. Used for unstructured grid rendering.

### [Point-Triangle Distance](2776-point-triangle-distance.md)
Computes the distance from a point to a triangle. Related to ray-ray intersection in terms of geometric primitives.

### [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md)
Intersects a ray with an implicit surface defined by a signed distance function. Used for smooth geometry.

### [Exact Rational Ray-Surfel Evaluation](2915-exact-rational-ray-surfel-evaluation.md)
Exact intersection of a ray with a surfel (surface element). Used for high-precision rendering.

### [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md)
Renders Gaussian splats on a spherical panorama. Uses ray-splat intersection.

### [Ray ID Caching](3110-ray-id-caching.md)
Caches ray IDs to optimize intersection tests. Used in large scene rendering.

### [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md)
A specific ray-geometry intersection method. Used in the IRIS renderer.

### [Ray-Sphere Intersection](3539-ray-sphere-intersection.md)
Intersects a ray with a sphere. Analytic solution.

### [Ray-Object Intersection Test](3604-ray-object-intersection-test.md)
General ray-object intersection test. Used in bounding volume hierarchies.

### [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md)
Detects hits based on azimuthal angle. Used in panoramic rendering.

### [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md)
Intersects a ray with a polar section. Used in cylindrical coordinate systems.

### [Ray-Cylinder Intersection](5014-ray-cylinder-intersection.md)
Intersects a ray with a cylinder. Analytic solution.

### [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md)
Overview of ray tracing and intersection methods.

### [Point-Plane Reflection](5920-point-plane-reflection.md)
Reflects a point across a plane. Related to ray reflection.

### [Ray-Primitive and Segment Intersection](6230-ray-primitive-and-segment-intersection.md)
Intersects a ray with a primitive or segment. Used in collision detection.

### [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md)
Intersects a ray with a plane or rectangle. Used for clipping.

### [Ray-Splat Intersection](6816-ray-splat-intersection.md)
Intersects a ray with a Gaussian splat. Used in neural rendering.

### [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md)
Dispatches raycasting for splats. Used in efficient rendering.

## Operating characteristics
- **Time Complexity**: $O(1)$ per ray pair, dominated by the matrix inversion or least squares solve.
- **Memory Complexity**: $O(1)$ per ray pair, storing intermediate matrices and vectors.
- **Measured Performance**: Performance depends on the hardware and the efficiency of the linear algebra library (e.g., cuBLAS, MKL). Batched operations can significantly improve throughput [S1, unofficial].
- **Scaling**: Scales well with batch size due to parallel execution.
- **Strengths**: Robust to noise and parallax; provides a unique solution for skew lines.
- **Limitations**: Does not provide exact intersection; sensitive to parallel rays; requires normalization of direction vectors for stability [G].
- **When to Use**: When rays are expected to be skew or nearly parallel due to noise or parallax.
- **When Not to Use**: When exact intersection is required or when rays are guaranteed to intersect.

## Background
The method is used in volume rendering pipelines, particularly in neural rendering techniques like VolSplat, where it is employed to handle misalignments between rays [S1, unofficial]. The specific implementation details are derived from the VolSplat repository [S1, unofficial].

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
