# LinPrim Differentiable Rendering Pipeline

Also called: LinPrim Rendering, LinPrim rasterizer, Linear Primitive Rendering, Linear Primitives for Differentiable Volumetric Rendering, Octahedron Rendering

## Overview
LinPrim is a differentiable rendering pipeline that replaces the standard 3D Gaussian kernel with convex polyhedral primitives, specifically octahedra, to enable efficient volumetric rendering via ray-triangle intersections [S1]. Unlike Gaussian Splatting which relies on 2D Gaussian projections and density fields, LinPrim computes opacity based on the path length of rays through the primitive's geometry, allowing gradients to be backpropagated directly through the intersection calculations [S1]. It retains the core 3DGS preprocessing and compositing structure but diverges in the primitive representation and the per-pixel evaluation logic, offering a bounded geometric alternative that supports real-time performance while maintaining high-fidelity reconstruction [S1][S2, unofficial].

## How it works
The pipeline operates in two stages: per-primitive preprocessing and per-pixel rasterization/compositing [S1].

### Inputs and Outputs
*   **Inputs**:
    *   A set of $M$ primitives. Each primitive $i$ is defined by:
        *   Position $\mathbf{p}_i \in \mathbb{R}^3$.
        *   Distances defining the shape (e.g., for octahedra, distances to vertices from the center) [S1].
        *   Rotation features $\mathbf{R}_i$ [S1].
        *   View-dependent color coefficients: 48 Spherical Harmonic (SH) coefficients (floats) per primitive [S1].
    *   Camera pose (extrinsic matrix) and intrinsic parameters [S1].
*   **Outputs**:
    *   A 2D image with color and opacity values for each pixel [S1].

### Data Structures
*   **Primitive Representation**:
    *   **Octahedron**: Defined by a center position, a rotation matrix, and a single distance parameter that describes the distance to all vertices (enforcing symmetry, where opposite vertices share the same distance) [S1]. This reduces complexity and ensures the center is the true geometric center [S1].
    *   **Tetrahedron**: An alternative primitive mentioned, requiring 12 floats to describe non-symmetric shapes [S1].
    *   **Attributes**: Each primitive stores 48 SH coefficients for view-dependent color [S1].
*   **Preprocessed Data**:
    *   **Bounding Box**: Axis-aligned bounding box (AABB) in screen space for each primitive, used for culling [S1].
    *   **L1-Slab Parameters**: For the octahedron, the projection stores parameters defining the L1-slab in the projected space [S2, unofficial].
    *   **Sorted List**: Primitives are sorted front-to-back within tiles [S1].

### Steps

1.  **Construction**:
    *   Construct polyhedra (octahedra or tetrahedra) from Structure-from-Motion (SfM) points in a fully differentiable manner [S1].
    *   For octahedra, enforce symmetry by using a single distance for opposing corners [S1].

2.  **Preprocessing** (Per-Primitive):
    *   Transform the primitive's position and rotation into the camera's coordinate system using the camera pose [S1].
    *   Calculate the view-dependent color using the viewing direction and the stored SH coefficients [S1].
    *   Project the primitive into the 3D affine ray space approximation (EWA Splatting) [S1]. This step approximates the 3D shape in camera space. The error from this approximation is negligible for small primitives [S1].
    *   Compute the projected bounding box in screen space to determine visibility [S1].
    *   *Note*: For the octahedron variant, the `project` function stores the L1-slab parameters of the projected octahedron [S2, unofficial].

3.  **Global Sorting and Tiling**:
    *   Tile the screen into regions [S1].
    *   Sort primitives within each tile according to an approximate front-to-back order [S1]. This allows for efficient compositing and early termination [S1].

4.  **Rasterization** (Per-Pixel):
    *   For each pixel, reconstruct the pixel ray using the +0.5 pixel-center convention [S2, unofficial].
    *   Determine if the ray intersects the convex primitive. Since primitives are convex, the ray intersects either twice or not at all [S1].
    *   Use the Möller-Trumbore Intersection Algorithm (MTIA) to find the intersection points [S1]. This algorithm is fully differentiable [S1].
    *   If two intersections exist, calculate the opacity based on the distance between the entry and exit points [S1].
    *   **Normalization**: To handle scale ambiguity, normalize the intersection distances. Determine the smallest distance $d$ among all primitives and scale all distance measurements accordingly [S1].
    *   **Opacity/Density Calculation**:
        Given the primitive opacity $\alpha$, the density $\sigma$ is defined as:
        $$ \sigma(\alpha) = -\frac{\log(1 - 0.99 \cdot \alpha)}{2 \cdot \min(d_x, d_y, d_z)} $$
        where $d_x, d_y, d_z$ are the normalized distances [S1].
        *Note*: The source [S2, unofficial] describes the evaluation as computing the Beer-Lambert opacity from the ray's entry and exit depths through the octahedron.

5.  **Compositing**:
    *   Aggregate color contributions of all visible primitives via alpha blending in front-to-back order [S1].
    *   Stop blending once the cumulative opacity reaches a threshold of 0.999 [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_linprim(primitives, camera, image_width, image_height):
    """
    Renders the scene using the LinPrim pipeline.
    primitives: List of Primitive objects.
    camera: Camera object with pose and intrinsics.
    """
    # 1. Preprocessing
    preprocessed_primitives = []
    for p in primitives:
        # Transform to camera space
        p_cam = transform_to_camera_space(p, camera.pose)
        
        # Calculate view-dependent color using SH
        # Note: SH evaluation depends on viewing direction, handled during rasterization or precomputed
        # Here we assume color is evaluated per-pixel based on direction
        
        # Project to screen space / Ray space
        # For Octahedron: Store L1-slab parameters
        # For general: Compute bounding box in screen space
        bbox = compute_screen_bbox(p_cam)
        
        preprocessed_primitives.append({
            'primitive': p,
            'bbox': bbox,
            'cam_data': p_cam
        })

    # 2. Global Sorting and Tiling
    tiles = create_tiles(image_width, image_height)
    for tile in tiles:
        # Cull primitives outside tile bbox
        tile_prims = [p for p in preprocessed_primitives if intersects_tile(p['bbox'], tile)]
        # Sort front-to-back (approximate)
        tile_prims.sort(key=lambda x: depth_estimate(x, camera))
        tile.primitives = tile_prims

    # 3. Rasterization and Compositing
    image = zeros((image_height, image_width, 3))
    opacity_map = zeros((image_height, image_width))
    
    # Find minimum distance for normalization (global pass or approximation)
    min_d = min(p['primitive'].min_dimension for p in preprocessed_primitives)

    for y in range(image_height):
        for x in range(image_width):
            # Reconstruct ray
            ray = reconstruct_ray(x, y, camera)
            
            # Get primitives for this pixel's tile
            tile = get_tile(x, y)
            cumulative_opacity = 0.0
            color = zeros(3)
            
            for prim_data in tile.primitives:
                p = prim_data['primitive']
                
                # Check if pixel is in bbox (fast cull)
                if not point_in_bbox((x, y), prim_data['bbox']):
                    continue
                
                # Ray-Primitive Intersection
                # Use Möller-Trumbore Algorithm for triangles forming the primitive
                # For Octahedron, this involves checking against its triangular faces
                intersections = ray_triangle_intersections(ray, p)
                
                if len(intersections) == 2:
                    d1, d2 = sorted(intersections)
                    path_length = d2 - d1
                    
                    # Normalize distance
                    # Note: Source says scale all measurements relative to smallest distance d
                    # Here we assume normalized_dist is derived from path_length and min_d
                    normalized_dist = normalize_distance(path_length, min_d)
                    
                    # Calculate Opacity/Density
                    # Source defines density sigma(alpha). 
                    # In standard alpha blending, we need opacity alpha for the step.
                    # The source implies alpha is a stored feature or derived.
                    # Assuming alpha is a stored attribute of the primitive for now,
                    # or derived from the density and path length.
                    # Standard Beer-Lambert: alpha = 1 - exp(-sigma * path_length)
                    # However, LinPrim uses the path length directly to determine opacity contribution.
                    # Let's assume the primitive has a base opacity alpha_base.
                    
                    alpha_base = p.opacity # Stored attribute
                    
                    # Calculate effective opacity for this step
                    # The source formula: sigma(alpha) = -log(1 - 0.99 * alpha) / (2 * min_dist)
                    # This seems to define the density required to achieve a target alpha.
                    # For rendering, we likely use the stored alpha and the path length.
                    # Let's assume the contribution is based on the path length through the volume.
                    
                    # Simplified interpretation:
                    # Opacity contribution is determined by the path length.
                    # Let's use the standard alpha blending with the primitive's alpha.
                    # The "density" formula might be for optimization/initialization.
                    
                    # To be safe and follow the "Beer-Lambert" mention in [S2, unofficial]:
                    # We need a sigma. If alpha is the target opacity for the whole primitive,
                    # and path_length is the max path, then sigma = -log(1-alpha)/path_length.
                    # But we have a specific path_length for this ray.
                    
                    # Let's assume the primitive stores 'alpha' as the max opacity.
                    # And we compute the opacity for this specific ray segment.
                    # If the primitive is homogeneous, the opacity for a segment of length L is:
                    # alpha_seg = 1 - exp(-sigma * L)
                    # where sigma is derived from the primitive's total alpha and max path.
                    
                    # However, the source [S1] says "opacity depends on the distance between intersection points".
                    # And provides a formula for sigma(alpha).
                    # Let's assume the primitive has a stored 'alpha' (e.g., 0.99).
                    # And we use the normalized distance to compute the local opacity.
                    
                    # Re-reading [S1]: "If there are two, the opacity depends on the distance between the intersection points."
                    # And "we define the density of an octahedron as: sigma(alpha) = ..."
                    # This suggests sigma is computed from the primitive's alpha and its size.
                    
                    sigma = -math.log(1 - 0.99 * alpha_base) / (2 * normalized_dist)
                    alpha_step = 1 - math.exp(-sigma * normalized_dist)
                    
                    # View-dependent color
                    # Evaluate SH at viewing direction
                    view_dir = ray.direction
                    color_prim = evaluate_sh(p.sh_coeffs, view_dir)
                    
                    # Alpha Blending
                    alpha_step_normalized = alpha_step * (1 - cumulative_opacity)
                    color += color_prim * alpha_step_normalized
                    cumulative_opacity += alpha_step_normalized
                    
                    if cumulative_opacity > 0.999:
                        break
            
            image[y, x] = color
            opacity_map[y, x] = cumulative_opacity

    return image

def ray_triangle_intersections(ray, primitive):
    """
    Returns a list of intersection distances for a ray and a convex polyhedron.
    Uses Möller-Trumbore algorithm for each face.
    """
    intersections = []
    for face in primitive.faces:
        t = moller_trumbore(ray.origin, ray.direction, face)
        if t is not None:
            intersections.append(t)
    return intersections

def moller_trumbore(origin, direction, face):
    """
    Standard Möller-Trumbore intersection algorithm.
    Returns distance t or None.
    """
    # Implementation of MTIA
    # ...
    pass

def normalize_distance(path_length, min_d):
    """
    Normalizes the path length relative to the minimum primitive distance.
    """
    # Source: "scaling all distance measurements accordingly"
    # Assuming min_d is the global minimum distance found in preprocessing
    return path_length / min_d
```

### Parameters
*   **SH Degree**: 48 coefficients (typically degree 2 or 3) per primitive [S1].
*   **Opacity Threshold**: 0.999 for stopping compositing [S1].
*   **Normalization Factor**: The smallest distance $d$ among all primitives in the scene [S1].

### Edge Cases and Pitfalls
*   **Scale Ambiguity**: Without normalization, the appearance of primitives varies across different scene sizes. This is handled by scaling distances relative to the smallest primitive distance $d$ [S1].
*   **Approximation Error**: The 3D affine ray space approximation error grows with distance from the center points. However, for small primitives (required for high-fidelity reconstruction), this error remains negligible [S1].
*   **Degenerate Input**: The method assumes convex primitives. Non-convex shapes would require more complex intersection logic [S1].
*   **Memory**: The majority of memory per primitive is used by the 48 SH coefficients. Reducing SH degree or primitive count is necessary for memory-constrained environments [S1].

## Operating characteristics
*   **Time Complexity**: Real-time performance is claimed, comparable to 3DGS [S1]. The two-stage process (preprocessing then rasterization) allows efficient parallelization [S1].
*   **Memory Complexity**: Dominated by the SH coefficients (48 floats per primitive) [S1].
*   **Strengths**:
    *   Fully differentiable through ray-triangle intersections [S1].
    *   Bounded geometry allows for intuitive modification and understanding [S1].
    *   Real-time rendering speeds [S1].
    *   High-fidelity reconstruction [S1].
*   **Limitations**:
    *   Scale ambiguity requires global normalization [S1].
    *   Memory intensive due to SH coefficients [S1].
    *   Approximation error in ray space projection, though negligible for small primitives [S1].
*   **When to use**: When bounded geometric primitives are preferred over Gaussian kernels, or when differentiable volumetric rendering with convex shapes is required [S1].

## Background
LinPrim was introduced as a method for differentiable volumetric rendering using linear primitives (octahedra and tetrahedra) [S1]. It builds on the 3DGS pipeline but replaces the Gaussian primitive with a polyhedron [S1]. The method was developed to provide an intuitive representation that can be easily understood and modified, while retaining real-time rendering speeds [S1]. It is cited in the context of point-based differentiable rendering extensions [S2, unofficial].

## Sources
[S1] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S2] XPR: An Extensible Cross-Platform Point-Based Differentiable Renderer, https://doi.org/10.48550/arxiv.2606.11529, fetched 2026-10-26 (unofficial)
