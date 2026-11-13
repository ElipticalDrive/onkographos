# Point-based Projected Tetrahedra

Also called: Hardware-Assisted Point-Based Volume Rendering, Point-Based Volume Rendering of Tetrahedral Meshes, Point-based footprint rendering

## Overview
Point-based Projected Tetrahedra (PBVR) is a technique for rendering large, unstructured tetrahedral meshes by approximating each tetrahedron with a single point primitive [S1]. It solves the problem of interactive visualization of unstructured data by eliminating the need to process or store connectivity information and avoiding convolution operations [S1]. Unlike sibling methods that reconstruct geometry or rely on complex sorting, PBVR decouples footprint generation and reshaping from compositing, using hardware shaders to reshape point footprints (e.g., to ellipsoids) and composite fragments directly [S1].

## How it works
**Inputs:** An unstructured tetrahedral mesh with scalar data (vertex-centered or cell-centered) and a camera viewpoint.
**Outputs:** A 2D image with color and opacity.

**Data Structures:**
1.  **Point List:** A list of point primitives. Each point contains:
    *   `v`: 3D position (centroid of the tetrahedron).
    *   `s`: Scalar value (mean of vertex values or cell value).
    *   `T`: 4x4 affine transformation matrix mapping the unit tetrahedron to the target tetrahedron.
    *   `n`: Normal vector (pre-computed gradient, optional for shading).
2.  **Pre-integration Table:** A 1D or 2D texture storing integrated opacity values for compositing [S1].

**Preprocessing (CPU):**
1.  For each tetrahedron $\sigma$ with vertices $P_0, P_1, P_2, P_3$:
    *   Define the unit tetrahedron $\hat{\sigma}$ inscribed in the unit sphere, centered at the origin.
    *   Compute the affine transformation $T$ such that $T(\hat{\sigma}) = \sigma$ [S1].
    *   Compute the centroid $v = T(\hat{c})$, where $\hat{c}$ is the centroid of $\hat{\sigma}$ (origin).
    *   Assign scalar $s$:
        *   If vertex-centered: $s = \frac{1}{4} \sum_{i=0}^{3} s_i$, where $s_i$ are scalar values at vertices [S1].
        *   If cell-centered: $s$ is the cell value [S1].
    *   (Optional) Compute gradient $\nabla s$ for normal $n$ [S1].
2.  Store the list of points with attributes $(v, s, T, n)$ [S1].

**Per-Frame Rendering:**

1.  **Sorting (CPU):** Sort the point list front-to-back based on the depth of $v$ relative to the current viewpoint [S1].
2.  **LOD Selection (CPU):** Determine the level-of-detail for the current frame. This may involve selecting a subset of points or adjusting point sizes based on distance to viewpoint and tetrahedron size [S1].
3.  **Vertex Shader (GPU):**
    *   For each point, use $T$ to determine the screen-space footprint.
    *   For **Ellipsoidal Footprints**: The min-volume enclosing ellipsoid of the tetrahedron is used. Since the unit sphere is the min-volume ellipsoid for the unit tetrahedron, $T$ maps the unit sphere to the target ellipsoid [S1].
    *   Pass the screen-space center and radius (or ellipsoid parameters) to the fragment shader via texture coordinates [S1].
    *   Pass the scalar $s$ and normal $n$ [S1].
4.  **Fragment Shader (GPU):**
    *   **Culling:** Discard fragments outside the footprint shape (e.g., if distance from center > radius for circular/ellipsoidal) [S1].
    *   **Compositing:**
        *   Maintain a depth buffer or use a texture to store the accumulated fragment [S1].
        *   Calculate the optical depth $\tau$ for the segment between the current fragment and the previously composited fragment.
        *   Use the pre-integration table to determine the opacity $\alpha$ for this segment based on $\tau$ and the scalar $s$ [S1].
        *   Composite the current fragment into the accumulated result using the Porter-Duff over operation [S1].
        *   Note: Compositing is done by fragment distances, not pre-assigned alpha values [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing
def preprocess_mesh(tetrahedra, scalar_data, data_type):
    points = []
    for tet in tetrahedra:
        # Compute affine transform T from unit tetrahedron to tet
        T = compute_affine_transform(tet.vertices)
        
        # Centroid of unit tetrahedron is origin (0,0,0)
        v = T @ vec3(0, 0, 0)
        
        # Assign scalar
        if data_type == 'vertex':
            s = mean(scalar_data[v_idx] for v_idx in tet.vertices)
        else:
            s = scalar_data[tet.id]
            
        # Optional: Compute normal
        n = compute_gradient(tet.vertices, scalar_data)
        
        points.append({'v': v, 's': s, 'T': T, 'n': n})
    return points

# Per-Frame
def render(points, camera, pre_int_table):
    # 1. Sort front-to-back
    sorted_points = sort_by_depth(points, camera.position)
    
    # 2. LOD Selection (simplified)
    active_points = select_lod(sorted_points, camera)
    
    # 3. GPU Rendering
    # Vertex Shader:
    # For each point p in active_points:
    #   screen_center, footprint_params = project_footprint(p.v, p.T, camera)
    #   output(screen_center, footprint_params, p.s, p.n)
    
    # Fragment Shader:
    # For each fragment f generated by footprint:
    #   if not inside_footprint(f, footprint_params): discard
    #   
    #   # Get accumulated depth/scalar from previous fragment in ray
    #   prev_depth, prev_alpha, prev_color = get_accumulated_fragment(f)
    #   
    #   # Calculate optical depth for segment
    #   segment_length = f.depth - prev_depth
    #   tau = calculate_optical_depth(segment_length, f.s)
    #   
    #   # Lookup opacity from pre-integration table
    #   alpha = pre_int_table.lookup(tau, f.s)
    #   
    #   # Composite
    #   new_color = f.color * alpha + prev_color * (1 - alpha)
    #   new_alpha = alpha + prev_alpha * (1 - alpha)
    #   
    #   # Update accumulated fragment
    #   update_accumulated_fragment(f, new_color, new_alpha, f.depth)
```

**Subtopics:**
*   **Ellipsoidal Footprints:** Uses the min-volume enclosing ellipsoid to reduce overdraw. Sets it apart by providing a tighter fit than circular footprints, improving image quality at the cost of more complex fragment culling [S1]. Choose when high fidelity is needed and overdraw is a concern [S1].
*   **Circular Footprints:** Uses a simple circle in screen space. Sets it apart by being computationally cheaper than ellipsoids but with higher overdraw [S1]. Choose for faster interaction or simpler hardware [S1].
*   **Direct Rendering (Sibling):** Renders tetrahedra as faces. Sets it apart by avoiding approximation error but generating excessive fragments for varying tetrahedron sizes [S1].

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ where $N$ is the number of tetrahedra [S1]. Rendering is dominated by fragment processing, which is $O(F)$ where $F$ is the number of fragments generated [S1].
*   **Memory Complexity:** $O(N)$ to store point attributes [S1].
*   **Performance:** Achieves interactive framerates for large datasets (e.g., 3.5M tetrahedra) [S1]. Performance scales with the number of fragments; ellipsoidal footprints reduce overdraw compared to circular or square footprints [S1].
*   **Strengths:** No connectivity processing; robust to malformed tetrahedra; easy LOD implementation; avoids convolution [S1].
*   **Limitations:** Approximation error; boundary quality degrades due to lack of scalar interpolation across the tetrahedron; sub-pixel tetrahedra are handled well, but large variations in tetrahedron size can lead to excessive fragment generation if not managed by LOD [S1].
*   **When to use:** Interactive visualization of large unstructured tetrahedral meshes [S1].
*   **When not to use:** When exact boundary representation is critical and approximation error is unacceptable; when hardware does not support programmable shaders for compositing [S1].

## Background
PBVR was introduced in a paper by the authors of [S1] to address the challenge of interactively visualizing large, unstructured tetrahedral meshes. It builds on the principles of point-based rendering and projected tetrahedra, aiming to combine the speed of point splatting with the accuracy of volume rendering. The method was applied to datasets such as Heart, SPX, fighter, and F16 [S1].

## Sources
[S1] Hardware-Assisted Point-Based Volume Rendering of Tetrahedral Meshes, https://www.sci.utah.edu/~eranders/papers/pbvr.pdf, fetched 2026-10-28
