# Point-based Projected Tetrahedra

Also called: Hardware-Assisted Point-Based Volume Rendering, Point-Based Volume Rendering of Tetrahedral Meshes, Point-based footprint rendering

## Overview
Point-based Projected Tetrahedra (PBVR) is a hardware-accelerated technique for interactively visualizing large, unstructured tetrahedral meshes by approximating each tetrahedron with a single point primitive [S1]. It solves the performance bottleneck of processing connectivity and sub-pixel primitives in direct rendering by decoupling footprint generation from compositing, using affine transformations to reshape point footprints into ellipsoids or circles to minimize overdraw [S1]. Unlike sibling methods that reconstruct geometry or use hierarchical partitioning, PBVR generates no additional geometry and avoids convolution operations by compositing fragments directly in hardware shaders [S1].

## How it works
**Inputs:** Unstructured tetrahedral mesh with vertex or cell-centered scalar data; camera viewpoint.
**Outputs:** 2D image with volume-rendered opacity and color.

**Data Structures:**
1.  **Point List:** An array of point primitives. Each point stores:
    *   `v`: 3D position (centroid of the tetrahedron).
    *   `s`: Scalar value (mean of vertex scalars or cell scalar).
    *   `T`: 4x4 affine transformation matrix mapping a unit tetrahedron to the target tetrahedron.
    *   `n`: Normal vector (pre-computed gradient, optional for shading).
2.  **Pre-integrated Transfer Function Table:** A 1D or 2D texture mapping scalar values and distances to color and opacity [S1].

**Preprocessing (CPU):**
1.  For each tetrahedron $\sigma$ with vertices $P_0, P_1, P_2, P_3$:
    *   Compute the centroid $v = \frac{1}{4}\sum_{i=0}^{3} P_i$.
    *   Compute the affine transformation $T$ that maps the unit tetrahedron $\hat{\sigma}$ (inscribed in the unit sphere, centered at origin) to $\sigma$. This involves solving for the linear part $M$ and translation $t$ such that $T(\hat{\sigma}) = \sigma$ [S1].
    *   Assign scalar $s$:
        *   If vertex-centered: $s = \frac{1}{4}\sum_{i=0}^{3} s(P_i)$.
        *   If cell-centered: $s = s_{cell}$ [S1].
    *   (Optional) Compute gradient $\nabla s$ at $v$ for shading normals [S1].
    *   Compute the minimum-volume enclosing ellipsoid for $\sigma$. Since the unit sphere is the min-volume ellipsoid for $\hat{\sigma}$, the transformation $T$ maps the unit sphere to the target ellipsoid. The ellipsoid parameters (center, axes) can be derived from $T$ [S1].

**Per-Frame Rendering:**
1.  **Sorting (CPU):** Sort the point list front-to-back based on the depth of $v$ relative to the current camera viewpoint [S1].
2.  **LOD Selection (CPU):** Determine the level-of-detail for each point based on distance to camera or screen coverage, resizing the point primitive accordingly [S1].
3.  **Vertex Shader (GPU):**
    *   For each point, apply $T$ to determine the screen-space footprint size and shape.
    *   Pass the footprint parameters (center, radius for circular; axes for ellipsoidal) to the fragment shader via texture coordinates or uniforms [S1].
    *   Output the scalar value $s$ and position [S1].
4.  **Fragment Shader (GPU):**
    *   **Culling:** For each fragment, determine if it lies within the approximating footprint (circle or ellipsoid). If outside, discard [S1].
    *   **Compositing:** Use the fragment's distance from the point center and the scalar value $s$ to look up color and opacity in the pre-integrated table. Composite the fragment into the frame buffer using the Porter-Duff over operation, accounting for the ray-gap between fragments [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing
def preprocess_tetrahedra(mesh):
    points = []
    for tet in mesh.tetrahedra:
        vertices = tet.vertices # [P0, P1, P2, P3]
        centroid = sum(vertices) / 4.0
        
        # Compute affine transform T from unit tetrahedron to this tet
        # Unit tet vertices are fixed: e.g., (1,1,1), (1,-1,-1), (-1,1,-1), (-1,-1,1) normalized
        # Solve for T such that T(unit_tet) = tet
        T = compute_affine_transform(unit_tet_vertices, vertices)
        
        # Assign scalar
        if mesh.data_type == 'vertex_centered':
            scalar = sum(mesh.vertex_scalars[v] for v in vertices) / 4.0
        else:
            scalar = mesh.cell_scalars[tet.id]
            
        # Compute min-volume enclosing ellipsoid parameters from T
        # The unit sphere maps to the ellipsoid via T
        ellipsoid_params = compute_ellipsoid_from_transform(T)
        
        points.append({
            'v': centroid,
            's': scalar,
            'T': T,
            'ellipsoid': ellipsoid_params
        })
    return points

def render(points, camera, screen_width, screen_height):
    # Sort points front-to-back
    sorted_points = sort_by_depth(points, camera.position)
    
    # Initialize frame buffer
    frame_buffer = initialize_frame_buffer(screen_width, screen_height)
    
    for point in sorted_points:
        # Determine LOD and screen-space footprint
        # Size depends on tetrahedron size and distance to camera
        footprint = compute_footprint(point, camera)
        
        # Rasterize footprint
        fragments = rasterize_footprint(footprint, screen_width, screen_height)
        
        for frag in fragments:
            # Check if fragment is inside the ellipsoid/circle
            if not is_inside_footprint(frag.position, footprint):
                continue
                
            # Get scalar value at fragment (interpolate if needed, but PBVR uses single scalar per tet)
            # PBVR uses the point's scalar s
            s = point['s']
            
            # Compute distance from center for pre-integration
            dist = distance(frag.position, footprint.center)
            
            # Lookup color and opacity
            color, opacity = pre_integrated_lookup(s, dist)
            
            # Composite
            frame_buffer = composite(frame_buffer, frag.position, color, opacity)
            
    return frame_buffer
```

**Edge Cases and Pitfalls:**
*   **Sub-pixel Tetrahedra:** Standard primitives are inefficient for sub-pixel areas; PBVR handles this well by using single points [S1].
*   **Boundary Quality:** Accuracy degrades at boundaries because single-point representation lacks scalar interpolation across the tetrahedron, limiting accurate pixel values when depth complexity is low [S1].
*   **Malformed Tetrahedra:** Unlike Hardware-Assisted Visibility Sorting (HAVS), PBVR remains robust and does not fail with malformed tetrahedra [S1].
*   **Overdraw:** Ellipsoidal footprints reduce overdraw compared to circular or square footprints, but exact projections render fewer fragments outside the true projection [S1].

## Subtopics

### [Projected Tetrahedra](877-projected-tetrahedra.md)
Renders tetrahedra as projected polygons, requiring connectivity processing and fragment sorting. Choose when exact geometric projection is needed and performance allows.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
Extracts isosurfaces from tetrahedral data. Choose for surface visualization rather than volume rendering.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
Optimizes projected tetrahedra for parallel execution using spatial partitioning. Choose for large-scale parallel rendering.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
Methods for storing tetrahedral connectivity. Choose when connectivity is required for other algorithms.

### [Delaunay Triangulation for Tetrahedral Representation](9905-delaunay-triangulation-for-tetrahedral-representation.md)
Generates tetrahedral meshes from point clouds. Choose for mesh generation.

### [TetraSDF](11632-tetrasdf.md)
Represents signed distance fields using tetrahedra. Choose for implicit surface representation.

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of tetrahedra. Per-frame sorting is $O(N \log N)$. Rendering is $O(F)$ where $F$ is the number of fragments generated.
*   **Memory Complexity:** $O(N)$ for storing point data and transforms.
*   **Performance:** Achieves interactive framerates for large datasets (e.g., 3.5M tetrahedra) [S1]. Performance scales with the number of fragments generated; ellipsoidal footprints reduce overdraw and improve performance compared to circular or square footprints [S1].
*   **Strengths:** No connectivity processing; robust to malformed tetrahedra; easy LOD implementation; interactive rates [S1].
*   **Limitations:** Boundary accuracy is lower due to lack of scalar interpolation; approximation error increases with footprint complexity [S1].
*   **When to use:** Interactive visualization of large, unstructured tetrahedral meshes where speed is critical and some boundary inaccuracy is acceptable [S1].
*   **When not to use:** When exact geometric projection is required or when high boundary accuracy is critical [S1].

## Background
PBVR was introduced by [S1] for the interactive visualization of large, unstructured tetrahedral meshes. It builds on the principles of point-based rendering and hardware-assisted compositing, extending them to tetrahedral volumes by using affine transformations to reshape point footprints. The method was developed to address the limitations of direct rendering and existing splatting techniques, which either require connectivity processing or convolution operations [S1]. It has been applied to datasets such as Heart, SPX, fighter, and F16 [S1].

## Sources
[S1] Hardware-Assisted Point-Based Volume Rendering of Tetrahedral Meshes, https://www.sci.utah.edu/~eranders/papers/pbvr.pdf, fetched 2026-10-28
