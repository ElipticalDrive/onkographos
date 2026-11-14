# SDF Value and Gradient Query

Also called: Live Cursor Measurement Caliper, ObjectFrameSDF query, Step Ruler Caliper, measureSdfRayCaliper

## Overview
This method provides real-time inspection of Signed Distance Field (SDF) properties at specific 3D coordinates, typically under a user cursor or for batched evaluation points. It outputs the signed distance value $d(\mathbf{p})$, the analytical gradient $\nabla d(\mathbf{p})$ (surface normal), and hit classification for a given point $\mathbf{p}$. It is distinguished from [SDF Value Computation](6443-sdf-value-computation.md) by its focus on interactive debugging and gradient verification, and from [Batched SDF Query with Joint Configuration](13037-batched-sdf-query-with-joint-configuration.md) by its explicit handling of world-to-object frame transformations and real-time ray-casting integration.

## How it works
**Inputs:**
1.  A set of query points $\mathbf{P} = \{\mathbf{p}_1, \dots, \mathbf{p}_N\}$ in world coordinates (or object frame, depending on implementation).
2.  An SDF primitive or mesh representation (e.g., `ObjectFrameSDF`, `VoxelGenerator`).
3.  Optional parameters: `compute_grad` (boolean), `resolution` (for grid-based queries), `query_range`.

**Outputs:**
1.  Signed distance values $d(\mathbf{p}_i)$.
2.  Gradient vectors $\nabla d(\mathbf{p}_i)$ (unit normals).
3.  Hit classification (e.g., COURSE SURFACE, GLASS MARBLE, ABYSS, SKY RAY) [S1, unofficial].
4.  Performance metrics (e.g., step counts, CSG latency) [S1, unofficial].

**Data Structures:**
*   **Query Points:** Array of shape $(N, 3)$ or $(B, N, 3)$ for batched inputs.
*   **SDF Values:** Array of shape $(N)$ or $(B, N)$.
*   **SDF Gradients:** Array of shape $(N, 3)$ or $(B, N, 3)$.
*   **Hit Class:** Enum or string indicating the nature of the ray intersection or free-space status [S1, unofficial].

**Steps:**
1.  **Coordinate Transformation:** If the SDF is defined in object space, transform world points $\mathbf{p}_{world}$ to object space $\mathbf{p}_{obj}$ using the inverse model matrix $M^{-1}$:
    $$ \mathbf{p}_{obj} = M^{-1} \mathbf{p}_{world} $$
    [S2, unofficial]
2.  **SDF Evaluation:** Query the SDF function $S(\mathbf{p})$ at the transformed points.
    *   If `compute_grad` is True, compute both value and gradient.
    *   If `compute_grad` is False, compute only the value for performance [S2, unofficial].
    $$ d = S(\mathbf{p}_{obj}) $$
3.  **Gradient Computation:**
    *   **Analytical/Gradient-based:** For mesh-based SDFs, the gradient is often derived from the closest point $\mathbf{c}$ on the mesh surface:
        $$ \nabla d(\mathbf{p}) = \frac{\mathbf{p} - \mathbf{c}}{\|\mathbf{p} - \mathbf{c}\|} $$
        The distance is $d = \|\mathbf{p} - \mathbf{c}\|$ (signed based on inside/outside) [S2, unofficial].
    *   **Verification:** Verify that $\|\nabla d(\mathbf{p})\| \approx 1.000$ [S1, unofficial].
4.  **Hit Classification:** Classify the point based on distance and ray properties:
    *   **COURSE SURFACE:** Point is near a coarse geometric approximation.
    *   **GLASS MARBLE:** Point is inside a specific material region.
    *   **ABYSS:** Point is deep inside the object.
    *   **SKY RAY:** Point is in free space with no intersection [S1, unofficial].
5.  **Free-Space Probe (Optional):** Apply a +1m offset to distance values for visualization purposes [S1, unofficial].
6.  **Performance Logging:** Record step counts (optimized vs. legacy) and CSG latency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_sdf_value_and_gradient(points_world, sdf_object, compute_grad=True):
    """
    Queries SDF value and gradient for points in world coordinates.
    
    Args:
        points_world: np.ndarray of shape (N, 3) or (B, N, 3)
        sdf_object: Object implementing __call__(points_obj) -> (dist, grad)
        compute_grad: bool, whether to compute gradients
        
    Returns:
        dist: np.ndarray of distances
        grad: np.ndarray of gradients (if compute_grad else None)
        hit_class: list of strings indicating hit type
    """
    # 1. Transform to object frame if necessary
    # Assuming sdf_object handles internal frame or points are already in object frame
    # If world-to-object transform is needed:
    # points_obj = sdf_object.world_to_object_transform(points_world)
    points_obj = points_world # Placeholder for actual transform logic [G]

    # 2. Query SDF
    # The SDF object returns distance and optionally gradient
    if compute_grad:
        dist, grad = sdf_object(points_obj)
    else:
        dist = sdf_object(points_obj, compute_grad=False)
        grad = None

    # 3. Verify gradient norm if computed
    if grad is not None:
        grad_norm = np.linalg.norm(grad, axis=-1, keepdims=True)
        # Normalize to ensure unit length
        has_direction = grad_norm > 0
        grad[has_direction] = grad[has_direction] / grad_norm[has_direction]
        # Verification step: assert ||grad|| == 1.000 [S1, unofficial]

    # 4. Classify hit target
    # This is application-specific logic based on distance and context [S1, unofficial]
    hit_class = classify_hit(points_obj, dist)

    return dist, grad, hit_class

def classify_hit(points, dist):
    """
    Simplified hit classification logic.
    """
    classes = []
    for d in dist:
        if d < -0.1:
            classes.append("ABYSS")
        elif d < 0.0:
            classes.append("GLASS MARBLE")
        elif d < 0.1:
            classes.append("COURSE SURFACE")
        else:
            classes.append("SKY RAY")
    return classes
```

**Parameters:**
*   `compute_grad`: Boolean. If False, gradient computation is skipped for speed [S2, unofficial].
*   `resolution`: Voxel cell size (e.g., 0.01) for grid-based queries [S2, unofficial].
*   `query_range`: Dimensions of the voxel grid [S2, unofficial].

**Edge Cases:**
*   **Out-of-Bounds:** Points outside the defined voxel grid or bounding box require a fallback strategy (e.g., ground truth SDF or bounding box approximation) [S2, unofficial].
*   **Zero Distance:** Points exactly on the surface may have undefined gradients; handle division by zero in normalization [S2, unofficial].
*   **Precision:** Floating-point errors in gradient normalization should be monitored [S1, unofficial].

## Subtopics

### Cached SDF
Accelerates SDF queries by storing values in a voxel grid, falling back to a ground truth SDF for points outside the cache [S2, unofficial]. A variant is TSDF (Truncated SDF), where `truncation_distance` creates a smaller voxel grid covering only the near-surface region; out-of-bounds points return the truncation value [S2, unofficial].
*   **Parameters:** `resolution` (voxel cell size, e.g., 0.01), `range_per_dim` (dimensions of the voxel grid, typically derived from the object's bounding box with padding), `out_of_bounds_strategy` (default compares against object bounding box; `pv.OutOfBoundsStrategy.LOOKUP_GT_SDF` uses the ground truth SDF), `truncation_distance` (distance threshold for truncation) [S2, unofficial].
*   **Pitfalls:** Bounding box comparison for out-of-bounds points always under-approximates the SDF value but is empirically sufficient for most applications [S2, unofficial].
*   **Comparison:** Cached SDF is dramatically faster than using the ground truth SDF for out-of-bounds queries [S2, unofficial].

### SDF Snapping
Queries the VoxelGenerator at floating-point positions to approximate the surface using SDF values, snapping instances closer to the surface [S3].
*   **Formula:** Position Snapping uses at least 2 nearby SDF samples to interpolate the surface distance: $P_{snapped} = P_{mesh} + \text{interpolation}(SDF(P_{mesh}), \text{samples})$ [S3].
*   **Pitfalls:** Does not work if the terrain is modified after initial generation, as generator estimations won't account for edited areas. Workaround: Set instances as persistent to save positions and avoid regeneration [S3]. Far Distance Artifacts: Snapping can cause instances to 'bury' or 'float' far away when low-resolution meshes are active, as they are moved closer to the high-resolution surface estimate [S3].
*   **Comparison:** Snapping improves positioning significantly but is slower. No snapping is faster but less precise. Snapping is preferably used when `lod_index > 0`, as LOD0 already has maximum precision [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ query points, assuming constant-time SDF evaluation. Cached SDF reduces average query time for in-cache points but adds lookup overhead [S2, unofficial]. SDF Snapping is slower than non-snapping due to interpolation and multiple samples [S3].
*   **Memory Complexity:** $O(N)$ for storing query results. Cached SDF requires $O(R^3)$ memory for the voxel grid, where $R$ is the resolution [S2, unofficial].
*   **Performance:** Real-time performance (e.g., 0.0018 ms CSG latency) is achievable for simple primitives [S1, unofficial]. Batched inference improves throughput by managing memory and reducing API calls [Parent].
*   **Strengths:** Real-time feedback for debugging, accurate gradient computation, flexible hit classification [S1, unofficial].
*   **Limitations:** Gradient computation can be expensive [S2, unofficial]. Snapping is sensitive to terrain modifications and low-resolution meshes [S3]. Out-of-bounds handling may introduce inaccuracies [S2, unofficial].

## Background
The method originates from interactive debugging tools for SDF-based rendering and physics, such as the "Live Cursor Measurement Caliper" used in real-time ray marching visualizations [S1, unofficial]. It is also used in volumetric structure evaluation for robotics and simulation, where object-frame SDF queries are necessary for chamfer distance calculations [S2, unofficial]. In game development, SDF snapping is used for instancing in voxel-based environments to ensure instances align with terrain surfaces [S3].

## Sources
[S1] feat(sdf-wizard): [W] 20-Second SDF X-Ray & Measurement Wizard (uDebugVizMode 1..5 + live caliper), https://github.com/bourkefloyd/gemini-4-argon-sfhackathon-2026/pull/23, fetched 2026-10-26 (unofficial)
[S2] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
[S3] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
