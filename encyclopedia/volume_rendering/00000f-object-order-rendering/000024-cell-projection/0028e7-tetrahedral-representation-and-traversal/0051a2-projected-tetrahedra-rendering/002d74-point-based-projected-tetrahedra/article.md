# Point-based Projected Tetrahedra

Also called: Hardware-Assisted Point-Based Volume Rendering, Point-Based Volume Rendering of Tetrahedral Meshes, Point-based footprint rendering

## Overview
Point-based Projected Tetrahedra (PBVR) is a technique for interactively visualizing large, unstructured tetrahedral meshes by approximating each tetrahedron with a single point primitive [S1]. It solves the problem of rendering sub-pixel sized primitives and the overhead of connectivity processing inherent in direct face-based rendering [S1]. Unlike its siblings, which rely on sorting or complex footprint generation, PBVR decouples footprint generation and reshaping from compositing, using affine transformations to reshape point footprints and hardware shaders for direct compositing, thereby avoiding convolution operations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An unstructured tetrahedral mesh with $N$ tetrahedra. Each tetrahedron has 4 vertices and associated scalar data (either vertex-centered or cell-centered).
*   **Output:** A 2D image with correct depth compositing.

**Data Structures:**
1.  **Point Representation:** Each tetrahedron $i$ is represented by a single point $v_i$.
    *   **Position:** The barycenter of the tetrahedron, transformed by the affine matrix $T_i$.
    *   **Scalar Value:**
        *   If vertex-centered: The mean of the scalar values at the 4 vertices.
        *   If cell-centered: The scalar value assigned to the cell.
    *   **Transformation Matrix:** A $4 \times 4$ affine transformation matrix $T_i$ that maps a regular unit tetrahedron $\hat{\sigma}$ (inscribed in the unit sphere, centered at the origin) to the target tetrahedron $\sigma_i$.
    *   **Normal (Optional):** Pre-computed gradient information from the scalar field for shading models [S1].

**Numbered Steps:**

1.  **Preprocessing (CPU):**
    For each tetrahedron $\sigma_i$:
    a.  Define the regular unit tetrahedron $\hat{\sigma}$ centered at the origin, inscribed in the unit sphere.
    b.  Compute the affine transformation $T_i$ such that $T_i(\hat{\sigma}) = \sigma_i$.
    c.  Compute the barycenter $c_i$ of $\sigma_i$.
    d.  Compute the representative point position $v_i = T_i(\hat{c})$, where $\hat{c}$ is the barycenter of $\hat{\sigma}$ (the origin). *Note: The source states "Transform the unit tetrahedron's barycenter by T to get vertex v". Since $\hat{\sigma}$ is centered at the origin, $v_i$ is effectively the centroid of $\sigma_i$.*
    e.  Assign the scalar value $s_i$ to $v_i$ (mean of vertex scalars or cell scalar).
    f.  (Optional) Compute gradients for normal information.

2.  **Per-Frame Sorting (CPU):**
    a.  Project the centroid $v_i$ of each point to screen space or compute depth relative to the viewpoint.
    b.  Sort all points front-to-back based on their depth.

3.  **Level-of-Detail (LOD) Selection (CPU):**
    a.  Determine the LOD for the current frame based on interaction state (compromise accuracy for speed during interaction, full quality after) or distance [S1].

4.  **Rendering (GPU):**
    a.  **Vertex Shader:** For each point $v_i$:
        i.  Retrieve the precomputed transformation $T_i$.
        ii. Calculate the point size based on the tetrahedron size and distance from the viewpoint.
        iii. Pass the center of the point and the radius (for circular) or ellipsoid parameters (for ellipsoidal) to the fragment shader via texture coordinates.
    b.  **Fragment Shader:**
        i.  **Culling:** Cull fragments based on the shape of the approximating element.
            *   *Circular:* Cull if distance from center > radius.
            *   *Ellipsoidal:* Cull if the fragment lies outside the minimum-volume enclosing ellipsoid of the tetrahedron. The ellipsoid is derived by applying $T_i$ to the unit sphere (the min-volume ellipsoid of $\hat{\sigma}$).
        ii. **Compositing:**
            i.  Keep one fragment in a texture.
            ii. Composite the ray-gap between fragments using a pre-integrated table.
            iii. Compositing is performed using fragment distances rather than pre-assigned alpha values to mitigate over-representation error [S1].

**Formulas:**
*   **Affine Transformation:** $T_i$ maps $\hat{\sigma}$ to $\sigma_i$.
*   **Conditional Probability (for Ellipsoid Selection):** $p(A|B) = \frac{s_A}{s_B}$, where $A$ is the tetrahedron, $B$ is the enclosing ellipsoid, $s_A$ is the surface area of $A$, and $s_B$ is the surface area of $B$. Minimizing $s_B$ maximizes the probability that a random ray hitting $B$ also hits $A$ [S1].
*   **Opacity:** $\alpha = 1 - \exp(-\tau_{avg} l)$ [S1, Parent].

**Parameters:**
*   **Point Size:** Determined by the size of the tetrahedron and the distance from the viewpoint [S1].
*   **Circular Footprint Parameters:** Center and radius in screen-space [S1].
*   **Ellipsoidal Footprint:** Defined by the min-volume enclosing ellipsoid [S1].

**Edge Cases and Pitfalls:**
*   **Sub-pixel Tetrahedra:** Standard primitives are inefficient for sub-pixel areas; PBVR handles this efficiently [S1].
*   **Boundary Quality:** Accuracy degrades at boundaries because single-point representation lacks scalar interpolation across the tetrahedron, limiting accurate pixel values when depth complexity is low [S1].
*   **Malformed Tetrahedra:** Unlike Hardware-Assisted Visibility Sorting (HAVS), PBVR remains robust and does not fail on malformed tetrahedra [S1].
*   **Wide Size Variation:** If tetrahedra sizes vary widely, point sizes for large tetrahedra increase rapidly, generating excessive fragments. Culling in the fragment program is expensive [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing
def preprocess_mesh(tetrahedra):
    points = []
    for tet in tetrahedra:
        # 1. Define unit tetrahedron sigma_hat centered at origin
        sigma_hat = get_unit_tetrahedron()
        
        # 2. Compute affine transform T mapping sigma_hat to tet
        T = compute_affine_transform(sigma_hat, tet)
        
        # 3. Compute representative point v (centroid of tet)
        v = T @ get_centroid(sigma_hat) # get_centroid(sigma_hat) is [0,0,0,1]
        
        # 4. Assign scalar value
        if tet.data_type == 'vertex_centered':
            s = mean(tet.vertex_scalars)
        else:
            s = tet.cell_scalar
            
        # 5. Compute min-volume enclosing ellipsoid parameters
        # The unit sphere is the min-volume ellipsoid for sigma_hat.
        # Apply T to the unit sphere to get the ellipsoid for tet.
        ellipsoid_params = compute_ellipsoid_from_transform(T)
        
        points.append({
            'position': v,
            'scalar': s,
            'transform': T,
            'ellipsoid_params': ellipsoid_params
        })
    return points

# Per-Frame Rendering
def render(points, viewpoint, screen_width, screen_height):
    # 1. Sort points front-to-back
    sorted_points = sort_front_to_back(points, viewpoint)
    
    # 2. Determine LOD (simplified)
    # In practice, this might filter points or adjust precision
    active_points = select_lod(sorted_points, viewpoint)
    
    # 3. GPU Rendering Pipeline
    fragments = []
    
    for point in active_points:
        # Vertex Shader
        center_screen = project_to_screen(point['position'], viewpoint)
        radius = calculate_point_size(point['position'], viewpoint)
        
        # Fragment Shader
        # Iterate over bounding box of the projected point
        for fragment in rasterize_square(center_screen, radius):
            # Ellipsoidal Culling
            if not is_inside_ellipsoid(fragment, point['ellipsoid_params']):
                continue
                
            # Compute depth and opacity
            depth = get_depth(fragment, point['position'], viewpoint)
            scalar = point['scalar']
            opacity = compute_opacity(scalar, depth)
            
            fragments.append({
                'depth': depth,
                'opacity': opacity,
                'color': transfer_function(scalar)
            })
            
    # 4. Compositing (Back-to-Front)
    # Note: The source mentions compositing ray-gaps using pre-integrated tables
    # and fragment distances. This pseudocode simplifies to standard over compositing
    # for clarity, as the exact pre-integration table lookup is hardware-specific.
    image = clear_image(screen_width, screen_height)
    for frag in sorted(fragments, key=lambda f: f['depth'], reverse=True):
        image = composite_over(image, frag['color'], frag['opacity'])
        
    return image
```

## Subtopics

### Parallelized Accurate Sorting for Projected Tetrahedra
Uses parallel sorting algorithms to handle the depth sorting requirement of PBVR and other projected methods more efficiently on the GPU. [Link](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)

### Sorting-free Pre-integrated Projected Tetrahedra
Eliminates the need for global sorting by using pre-integrated textures and specific footprint strategies to allow front-to-back compositing without explicit depth sorting. [Link](7491-sorting-free-pre-integrated-projected-tetrahedra.md)

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of tetrahedra. Per-frame sorting is $O(N \log N)$ on the CPU. Rendering is $O(F)$ where $F$ is the number of fragments generated, which depends on the footprint shape and LOD.
*   **Memory Complexity:** $O(N)$ to store the point representations and transformation matrices.
*   **Measured Performance:** Achieves interactive framerates on large datasets (e.g., Heart, SPX, fighter, F16) [S1].
*   **Strengths:**
    *   Eliminates connectivity processing and storage overhead [S1].
    *   Robust to malformed tetrahedra [S1].
    *   Easy implementation of LOD strategies [S1].
    *   Avoids convolution operations used in splatting [S1].
*   **Limitations:**
    *   Approximation error due to single-point representation [S1].
    *   Boundary quality degradation due to lack of scalar interpolation [S1].
    *   Excessive fragment generation if tetrahedra sizes vary widely [S1].
    *   Accuracy scales with approximation complexity; higher fidelity (e.g., ellipsoidal) increases fragment generation and decreases performance [S1].
*   **When to Use:** Interactive visualization of large, unstructured tetrahedral meshes where connectivity overhead is prohibitive and robustness to mesh quality is required.
*   **When Not to Use:** When high-fidelity boundary rendering is critical, or when tetrahedra sizes vary extremely widely, leading to excessive overdraw and fragment culling costs.

## Background
PBVR was introduced by [S1] to address the challenges of interactively visualizing large, unstructured tetrahedral meshes. It builds on the principles of point-based rendering and projected tetrahedra, leveraging GPU hardware for reshaping and compositing. The method was developed as part of research funded by the Department of Energy, NSF, and national laboratories, aiming to provide fast, effective visualization for scientific datasets [S1].

## Sources
[S1] Hardware-Assisted Point-Based Volume Rendering of Tetrahedral Meshes, https://www.sci.utah.edu/~eranders/papers/pbvr.pdf, fetched 2026-10-28
