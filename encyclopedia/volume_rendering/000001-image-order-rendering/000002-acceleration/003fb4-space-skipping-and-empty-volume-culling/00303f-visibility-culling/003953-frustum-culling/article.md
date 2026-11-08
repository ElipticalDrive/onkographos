# Frustum Culling
Also called: ToF View Frustum Culling, View Frustum Culling, markVisible
## Overview
Frustum culling is a visibility culling technique that avoids rendering geometry or evaluating volume samples that lie outside the camera's field of view, thereby eliminating unnecessary computation [S2, unofficial]. It is distinguished from its siblings by operating on the geometric relationship between the viewing frustum and scene primitives (bounding volumes, voxels, or points) rather than on opacity, occlusion, or probabilistic error estimates [S2, unofficial][S4, unofficial]. Unlike [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) which uses depth/opacity buffers, or [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) which relies on transmittance, frustum culling is a purely geometric pre-filter [S2, unofficial].
## How it works
**Inputs and Outputs:**
- **Inputs:** A set of scene primitives (points, voxels, blocks, or segments) defined by their positions and bounding volumes; the camera's view matrix, projection matrix, near plane distance ($n$), and far plane distance ($f$).
- **Outputs:** A boolean mask or list indicating which primitives are visible within the frustum [S1, unofficial][S2, unofficial].

**Data Structures:**
- **Bounding Volumes:** Each primitive is represented by a conservative bounding volume, typically a sphere or axis-aligned box (AABB) [S2, unofficial][S4, unofficial]. For spheres, the structure stores center $c_k$ and radius $r_k$ [S4, unofficial].
- **Frustum Planes:** The viewing frustum is defined by six planes (left, right, bottom, top, near, far) derived from the view and projection matrices [S2, unofficial].

**Steps:**
1. **Frustum Construction:** Compute the six clipping planes of the view frustum from the view and projection matrices [S2, unofficial].
2. **Bounding Volume Test:** For each primitive $k$ with bounding volume $B_k$, test if $B_k$ intersects the frustum [S2, unofficial].
   - For a sphere with center $c_k$ and radius $r_k$, the sphere is culled if it lies entirely outside any of the six planes [S4, unofficial].
   - The signed distance $d$ from the sphere center to a plane is computed. If $d < -r_k$ for any plane, the sphere is outside the frustum and is culled [S4, unofficial].
   - For an AABB, the test checks if the box is completely on the negative side of any plane [S2, unofficial].
3. **Hierarchical Acceleration (Optional):** If the scene is organized in a hierarchical spatial partition (e.g., a tree), test the bounding volume of a node against the frustum [S2, unofficial].
   - If a node is totally outside, all its descendants are culled [S2, unofficial].
   - If a node is totally inside, all its descendants are marked visible [S2, unofficial].
   - If the node intersects, recurse to children [S2, unofficial].
4. **Output Generation:** Return a boolean tensor or list of visible primitives [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_sphere_visible(center, radius, frustum_planes):
    """
    Check if a sphere is inside the frustum defined by 6 planes.
    frustum_planes: list of 6 tuples (normal, distance) where normal is unit vector.
    Returns True if visible (intersects or inside), False if culled.
    """
    for normal, dist in frustum_planes:
        # Signed distance from center to plane
        d = dot(normal, center) - dist
        # Cull if sphere is strictly outside the plane
        if d < -radius:
            return False
    return True

def frustum_cull_primitives(primitives, view_matrix, proj_matrix, near, far):
    """
    primitives: list of objects with 'center' (vec3) and 'radius' (float)
    Returns: list of primitives that are visible
    """
    # Construct frustum planes from view and projection matrices
    # This is a standard matrix decomposition; details omitted as per [S2, unofficial]
    frustum_planes = compute_frustum_planes(view_matrix, proj_matrix, near, far)
    
    visible_primitives = []
    for p in primitives:
        if is_sphere_visible(p.center, p.radius, frustum_planes):
            visible_primitives.append(p)
    return visible_primitives
```

**Parameters:**
- **Bounding Volume Type:** Sphere is common for its simplicity and rotational invariance [S4, unofficial]. AABB is common for voxel grids [S2, unofficial].
- **Hierarchical Structure:** Not always used; depends on the scene representation [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in plane distance calculations can cause jitter or incorrect culling for primitives near the frustum boundary [G].
- **Degenerate Input:** If the camera is inside a bounding volume, the standard "outside" test may fail if not handled correctly (e.g., sphere always intersects if center is inside) [G].
- **Tightness:** Conservative bounding volumes may admit primitives that are technically outside the frustum but inside the bounding volume, leading to false positives (over-culling is rare, under-culling is common) [S4, unofficial].

## Subtopics

### View Frustum Culling for Segments
An optimization technique used in segment-based tetrahedral meshing to reject segments outside the view frustum early in the rendering pipeline [S3]. It counters over-refinement caused by DAG dependencies, where regions outside the view frustum are unnecessarily refined, increasing mesh size [S3]. By detecting and rejecting these segments early, the system reduces the number of tetrahedra processed, improving performance [S3].

### Two-Level Visibility Culling
A hierarchical approach used in TideGS for scalable 3D Gaussian Splatting [S4, unofficial].
- **Level 1 (CPU):** Computes an active block set $\mathcal{K}_t$ for a camera batch $\mathcal{B}_t$ using coarse bounding spheres [S4, unofficial]. For each block $k$ with center $c_k$ and radius $r_k$, a 6-plane frustum test is applied [S4, unofficial]. A block is kept if its sphere intersects the frustum [S4, unofficial].
- **Level 2 (GPU):** Materializes selected resident blocks $R_t$ in VRAM and runs the standard 3DGS projection/rasterization pipeline [S4, unofficial]. The final contributing set $I_t$ is determined, and only Gaussians in $I_t$ participate in forward/backward passes [S4, unofficial].
- **Formula:** $\mathcal{K}_t = \bigcup_{c \in \mathcal{B}_t} \{k \mid \text{visible}(k, c)\}$ [S4, unofficial]. A sphere is culled if signed distance to plane $d < -r_k$ [S4, unofficial].
- **Comparison:** Level 1 is conservative (may admit extra blocks), while Level 2 applies exact 3DGS pipeline semantics, preserving standard rendering behavior while allowing block-level storage [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for non-hierarchical culling, where $N$ is the number of primitives [S2, unofficial]. With hierarchical spatial partitioning, it can be sub-linear, $O(\log N)$ or better, depending on the tree structure and scene distribution [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory for non-hierarchical culling [S2, unofficial]. Hierarchical culling requires storing the spatial partition structure [S2, unofficial].
- **Performance:** Significantly reduces the number of primitives processed by the GPU, especially in large scenes where only a small fraction of the geometry is visible [S2, unofficial][S4, unofficial].
- **Strengths:** Simple to implement, effective for large-scale scenes, reduces GPU load [S2, unofficial][S4, unofficial].
- **Limitations:** Conservative bounding volumes may lead to false positives (rendering some invisible geometry) [S4, unofficial]. Does not account for occlusion by other geometry [S2, unofficial].
- **When to Use:** When the scene is large and the camera view is limited, and when primitives have well-defined bounding volumes [S2, unofficial].
- **When Not to Use:** When the scene is small and fully visible, or when occlusion is the primary bottleneck [S2, unofficial].

## Background
Frustum culling is a fundamental technique in computer graphics, originally developed for real-time rendering of 3D scenes [S2, unofficial]. It is widely used in volume rendering to avoid processing voxels or samples that do not contribute to the final image [S2, unofficial]. The technique is discussed in various graphics literature, including Møller and Haines' work on volume/frustum intersection algorithms [S2, unofficial]. It is also applied in modern out-of-core rendering systems, such as TideGS, to manage large-scale 3D Gaussian Splatting models [S4, unofficial].

## Sources
[S1] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
[S3] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
[S4] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
