# SDF Value and Gradient Query

Also called: Live Cursor Measurement Caliper, ObjectFrameSDF query, Step Ruler Caliper, measureSdfRayCaliper

## Overview
SDF Value and Gradient Query is a method for inspecting Signed Distance Field (SDF) properties at specific 3D coordinates in real-time, typically for debugging, calibration, or precise spatial interaction. It outputs the signed distance $d(\mathbf{p})$, the analytical normal $\nabla d(\mathbf{p})$, and hit classification for a given point $\mathbf{p}$. It differs from [SDF Value Computation](6443-sdf-value-computation.md) by focusing on interactive, single-point or sparse inspection rather than bulk volume evaluation, and from [Batched SDF Query with Joint Configuration](13037-batched-sdf-query-with-joint-configuration.md) by handling static object frames rather than articulated joint hierarchies.

## How it works
The method supports two primary implementation patterns: interactive ray-casting inspection (caliper) and programmatic object-frame queries.

### Inputs and Outputs
*   **Inputs**: A 3D point $\mathbf{p}$ in world or object coordinates; a boolean flag `compute_grad` (default `True`).
*   **Outputs**:
    *   Signed distance $d(\mathbf{p}) \in \mathbb{R}$.
    *   Gradient/Normal $\nabla d(\mathbf{p}) \in \mathbb{R}^3$.
    *   Hit classification (e.g., COURSE SURFACE, GLASS MARBLE, ABYSS, SKY RAY) [S1, unofficial].

### Data Structures
*   **Raycasting Scene**: A spatial acceleration structure (e.g., `o3d.t.geometry.RaycastingScene`) storing triangle meshes [S2, unofficial].
*   **Query Points**: A tensor or array of shape $(N, 3)$ or $(B, N, 3)$ for batched queries [S2, unofficial].

### Steps

1.  **Point Acquisition**:
    *   *Interactive*: Cast a ray from the camera through the mouse cursor to intersect the scene. Determine the 3D world point $\mathbf{p}$ [S1, unofficial].
    *   *Programmatic*: Generate query points using a grid resolution and query range, or accept user-defined coordinates [S2, unofficial].

2.  **Distance and Gradient Computation**:
    *   Query the SDF representation at $\mathbf{p}$.
    *   If `compute_grad` is `True`, compute the gradient. For mesh-based SDFs, the gradient is derived from the vector from the query point to the closest point on the surface [S2, unofficial]:
        $$ \mathbf{g} = \mathbf{p}_{closest} - \mathbf{p} $$
        $$ \nabla d(\mathbf{p}) = \frac{\mathbf{g}}{\|\mathbf{g}\|} \quad \text{if } \|\mathbf{g}\| > 0 $$
    *   If `compute_grad` is `False`, skip gradient calculation to improve speed [S2, unofficial].

3.  **Verification and Classification**:
    *   Verify the normal magnitude: $\|\nabla d(\mathbf{p})\| \approx 1.000$ [S1, unofficial].
    *   Classify the hit target based on material properties or depth (e.g., COURSE SURFACE, ABYSS) [S1, unofficial].
    *   Apply a free-space probe offset (e.g., +1m) if required for visualization [S1, unofficial].

4.  **Performance Analysis (Optional)**:
    *   Compare optimized marching steps against legacy steps to calculate percentage reduction in steps, often utilizing bounds and $\kappa$-March optimizations [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_sdf_value_and_gradient(mesh, point, compute_grad=True):
    """
    Queries the SDF value and gradient for a single point.
    
    Args:
        mesh: A triangle mesh object supporting raycasting or closest point queries.
        point: A numpy array or tensor of shape (3,) representing [x, y, z].
        compute_grad: Boolean flag. If False, gradient computation is skipped.
        
    Returns:
        distance: Float, the signed distance.
        gradient: Numpy array of shape (3,), the normalized gradient/normal.
        hit_info: String or enum, classification of the hit target.
    """
    # 1. Find closest point on the mesh surface
    # Implementation depends on the backend (e.g., Open3D, custom BVH)
    # Assuming a generic closest_point query that returns the closest surface point
    closest_point = mesh.get_closest_point(point)
    
    # 2. Compute raw gradient vector
    raw_gradient = closest_point - point
    distance = np.linalg.norm(raw_gradient)
    
    # 3. Handle edge cases
    if distance < 1e-6:
        # Point is on the surface or inside; gradient is undefined or zero
        # [G] Default to surface normal if available, else zero vector
        gradient = np.zeros(3)
        if hasattr(mesh, 'get_normal_at'):
            gradient = mesh.get_normal_at(closest_point)
        hit_info = "SURFACE"
    else:
        # 4. Normalize gradient
        gradient = raw_gradient / distance
        
        # 5. Verify unit length (for debugging)
        # assert np.isclose(np.linalg.norm(gradient), 1.0, atol=1e-3)
        
        # 6. Classify hit
        # [G] Classification logic depends on specific scene materials
        hit_info = classify_hit(closest_point, point)

    # 7. Apply free-space probe offset if needed for visualization
    # display_distance = distance + 1.0 
    
    return distance, gradient, hit_info

def batched_query_sdf(mesh, points, compute_grad=True):
    """
    Queries SDF for a batch of points.
    
    Args:
        points: Tensor or array of shape (N, 3) or (B, N, 3).
        
    Returns:
        sdf_vals: Array of shape (N,) or (B, N).
        sdf_grads: Array of shape (N, 3) or (B, N, 3).
    """
    # Flatten for processing if batched
    original_shape = points.shape
    points_flat = points.reshape(-1, 3)
    
    n = points_flat.shape[0]
    sdf_vals = np.zeros(n)
    sdf_grads = np.zeros((n, 3))
    
    for i in range(n):
        d, g, _ = query_sdf_value_and_gradient(mesh, points_flat[i], compute_grad)
        sdf_vals[i] = d
        sdf_grads[i] = g
        
    # Reshape back to original batch structure
    sdf_vals = sdf_vals.reshape(original_shape[:-1])
    sdf_grads = sdf_grads.reshape(original_shape)
    
    return sdf_vals, sdf_grads
```

### Parameters
*   `compute_grad`: Boolean. If `False`, gradient computation is skipped for speed [S2, unofficial].
*   `free_space_offset`: Float. Default 1.0m, used for visualization probes [S1, unofficial].

### Edge Cases and Pitfalls
*   **Zero Distance**: If the query point is exactly on the surface, the gradient vector is zero. The implementation must handle this by returning a zero vector or a pre-computed surface normal [S2, unofficial].
*   **Precision**: Normalization errors can accumulate; verification of $\|\nabla d\| = 1.000$ is recommended for debugging [S1, unofficial].
*   **CSG Latency**: Real-time CSG operations may introduce latency (e.g., 0.0018 ms) which should be monitored [S1, unofficial].

## Subtopics

### [Cached SDF](13075-cached-sdf.md)
Cached SDF accelerates queries by storing quantized distance values in a voxel grid, falling back to a ground truth SDF for points outside the cache. It differs from standard SDF primitives by trading far-field precision for significant memory savings and query speed, using Int8 quantization to compress the field.

### SDF Snapping
SDF Snapping queries the VoxelGenerator at floating-point positions to approximate the surface, snapping instances closer to the surface [S3]. It uses at least 2 nearby SDF samples to interpolate the surface distance: $P_{snapped} = P_{mesh} + \text{interpolation}(SDF(P_{mesh}), \text{samples})$ [S3]. It is slower than non-snapping but improves positioning significantly, preferably used when `lod_index > 0` [S3]. Pitfalls include failure to work if terrain is modified after initial generation (workaround: set instances as persistent) and far-distance artifacts where instances may "bury" or "float" when low-resolution meshes are active [S3].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per point for pre-computed SDFs; $O(\log N)$ or $O(N)$ for mesh-based closest point queries depending on acceleration structure [S2, unofficial].
*   **Memory Complexity**: $O(N)$ for storing query points and results [S2, unofficial].
*   **Performance**: Real-time inspection is feasible (e.g., 60 FPS) with optimized raycasting [S1, unofficial]. CSG latency is minimal (0.0018 ms) for simple scenes [S1, unofficial].
*   **Strengths**: High precision for specific points; useful for debugging and calibration.
*   **Limitations**: Not suitable for bulk volume rendering; mesh-based queries can be slow for complex geometries without acceleration [S3].

## Background
The method originated from interactive debugging tools for SDF-based rendering, such as the "Live Cursor Measurement Caliper" used in SDF X-Ray visualizations [S1, unofficial]. It has been adopted in volumetric structure libraries for object-frame queries [S2, unofficial] and in voxel-based instancing systems for terrain alignment [S3].

## Sources
[S1] feat(sdf-wizard): [W] 20-Second SDF X-Ray & Measurement Wizard (uDebugVizMode 1..5 + live caliper), https://github.com/bourkefloyd/gemini-4-argon-sfhackathon-2026/pull/23, fetched 2026-10-26 (unofficial)
[S2] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
[S3] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
