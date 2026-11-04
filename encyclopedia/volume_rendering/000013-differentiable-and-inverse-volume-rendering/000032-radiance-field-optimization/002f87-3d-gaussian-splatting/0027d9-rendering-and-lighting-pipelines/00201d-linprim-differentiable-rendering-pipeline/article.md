# LinPrim Differentiable Rendering Pipeline

Also called: LinPrim Rendering, LinPrim rasterizer, Linear Primitives Rendering, Linear Primitives for Differentiable Volumetric Rendering

## Overview
LinPrim is a differentiable rendering pipeline that replaces the Gaussian ellipsoids of 3D Gaussian Splatting with convex polyhedra (specifically octahedra and tetrahedra) to enable efficient, real-time volumetric rendering with bounded geometry [S1]. It solves the problem of reconstructing scenes with sharp geometric features and fine details by allowing gradients to backpropagate through explicit ray-triangle intersections, distinguishing it from sibling methods that rely on implicit density fields or Gaussian kernels [S1]. Unlike [Duplex-GS](1988-duplex-gs.md) which uses dual representations, or [GsplatVolumeLighting](2203-gsplatvolumelighting.md) which focuses on lighting integration, LinPrim explicitly constructs and rasterizes transparent polyhedral volumes [S1].

## How it works
The pipeline operates in two main stages: per-primitive preprocessing and per-pixel rasterization/compositing [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Scene primitives defined by position, distances (vertex offsets), and rotation features [S1].
    *   Camera pose (view and projection matrices) [S1].
    *   Spherical Harmonic (SH) coefficients for view-dependent color [S1].
*   **Outputs:**
    *   Final rendered image with accumulated color and opacity [S1].

### Data Structures
Each primitive stores the following parameters [S1]:
1.  **Geometric Features:**
    *   Center position ($\mathbf{c}$) [S1].
    *   Distances from center to vertices. For octahedra, symmetry is enforced such that opposite vertices share the same distance, reducing complexity [S1]. For tetrahedra, 12 floats are required due to lack of full symmetry [S1].
    *   Rotation features (quaternion or rotation matrix) [S1].
2.  **Color Features:**
    *   48 Spherical Harmonic (SH) coefficients (floats) to capture view-dependent reflectance [S1]. This accounts for the majority of memory per primitive [S1].

### Numbered Steps

1.  **Primitive Construction:** Construct polyhedra (octahedra or tetrahedra) from Structure-from-Motion (SfM) points in a fully differentiable manner [S1]. Enforce symmetry for octahedra so that opposite vertices share the same distance from the center, ensuring the center is the true geometric center [S1].
2.  **Preprocessing (Per-Primitive):**
    *   Transform primitives into the camera's coordinate system using the camera pose [S1].
    *   Calculate view-dependent color based on the viewing direction using the stored SH coefficients [S1].
    *   Project primitives into the 3D affine ray space approximation (EWA Splatting) [S1]. This approximation error is negligible for small primitives [S1].
    *   Compute each primitive's projected bounding box in screen space to determine visibility [S1].
3.  **Global Sorting and Tiling:**
    *   Tile the screen into regions [S1].
    *   Sort primitives within each tile according to an approximate front-to-back order, leveraging techniques from 3DGS [S1].
4.  **Rasterization (Per-Pixel):**
    *   For each pixel ray, determine intersection with the convex primitive [S1].
    *   Use the Möller-Trumbore intersection algorithm (MTIA) to find intersections [S1]. This algorithm is fully differentiable [S1].
    *   If no intersections exist, the primitive is not visible for that pixel [S1].
    *   If two intersections exist (entry and exit), calculate the distance between them [S1].
5.  **Opacity and Density Calculation:**
    *   Normalize intersection distances to handle scale ambiguity. Determine the smallest distance $d$ among all primitives and scale all distance measurements accordingly [S1].
    *   Calculate density $\sigma$ based on the primitive opacity $\alpha$ and the normalized distances $d_x, d_y, d_z$ [S1]. The formula for an octahedron is:
        $$ \sigma(\alpha) = -\frac{\log(1 - 0.99 \cdot \alpha)}{2 \cdot \min(d_x, d_y, d_z)} $$
        where $\alpha$ is the primitive opacity and $d_x, d_y, d_z$ are the normalized distances [S1].
    *   Note: The sources state that optimizing for unnormalized densities directly was experimented with but found to decrease interpretability without performance gains [S1].
6.  **Compositing:**
    *   Aggregate color contributions of all visible primitives via alpha blending in front-to-back order [S1].
    *   Stop blending once the cumulative opacity reaches a threshold of 0.999 [S1].
7.  **Optimization (Backpropagation):**
    *   Backpropagate image-space errors through the intersection calculations (MTIA) to distribute gradients onto the geometric features (position, distances, rotation) of each polyhedron [S1].
    *   Adjust primitives' features and population based on gradient flow by comparing the rendered image with a known view [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linprim_render(primitives, camera_pose, image_width, image_height):
    """
    Renders the scene using the LinPrim pipeline.
    
    primitives: List of Primitive objects.
    camera_pose: Camera view and projection matrices.
    """
    
    # 1. Preprocessing
    transformed_primitives = []
    for prim in primitives:
        # Transform to camera space
        prim_cam = transform_to_camera_space(prim, camera_pose)
        
        # Calculate view-dependent color using SH
        # Note: SH evaluation depends on viewing direction, handled during rasterization or precomputed
        prim_cam.sh_coeffs = prim.sh_coeffs
        
        # Project to 3D affine ray space (EWA approximation)
        prim_cam.ray_space = project_to_ray_space(prim_cam)
        
        # Compute screen-space bounding box
        prim_cam.bbox = compute_bounding_box(prim_cam, image_width, image_height)
        
        transformed_primitives.append(prim_cam)

    # 2. Global Sorting and Tiling
    # Tile screen and sort primitives front-to-back within tiles
    tiles = create_tiles(image_width, image_height)
    for tile in tiles:
        # Get primitives overlapping this tile's bbox
        tile_prims = [p for p in transformed_primitives if intersects_bbox(p, tile.bbox)]
        # Sort by approximate depth (e.g., center depth)
        tile_prims.sort(key=lambda p: p.center_depth)
        tile.primitives = tile_prims

    # 3. Rasterization and Compositing
    # Determine scale normalization factor (smallest distance d)
    min_dist = min(p.get_min_distance() for p in transformed_primitives)
    
    image = zeros((image_height, image_width, 3))
    opacity_accum = zeros((image_height, image_width))
    
    for tile in tiles:
        for prim in tile.primitives:
            # Check if primitive is already fully opaque for this tile region
            # (Optimization: skip if opacity_accum >= 0.999)
            
            # Rasterize primitive onto pixels in tile
            for pixel in tile.pixels:
                if opacity_accum[pixel.y, pixel.x] >= 0.999:
                    continue
                
                # Ray-triangle intersection (Möller-Trumbore)
                # The primitive is a convex polyhedron (e.g., octahedron)
                # Find entry and exit points
                intersections = ray_intersect_polyhedron(pixel.ray, prim)
                
                if len(intersections) == 2:
                    d_entry, d_exit = intersections
                    dist = d_exit - d_entry
                    
                    # Normalize distance
                    normalized_dist = dist / min_dist
                    
                    # Get opacity alpha from primitive parameters
                    alpha = prim.opacity
                    
                    # Calculate density for this specific intersection segment
                    # Using the formula provided in sources
                    # Note: dx, dy, dz are normalized distances along axes or derived from dist
                    # The source formula uses min(dx, dy, dz). 
                    # In implementation, this might be approximated by normalized_dist or axis-aligned extents.
                    # Assuming normalized_dist represents the relevant scale metric here.
                    dx, dy, dz = normalized_dist, normalized_dist, normalized_dist 
                    sigma = -log(1 - 0.99 * alpha) / (2 * min(dx, dy, dz))
                    
                    # Compute color at intersection
                    # Interpolate color between entry and exit or use entry point
                    color = evaluate_sh(prim.sh_coeffs, pixel.view_direction)
                    
                    # Alpha blending
                    # C_out = C_in * (1 - alpha_eff) + C_prim * alpha_eff
                    # Here alpha_eff is derived from sigma and dist
                    alpha_eff = 1 - exp(-sigma * dist)
                    
                    image[pixel.y, pixel.x] = (
                        image[pixel.y, pixel.x] * (1 - alpha_eff) +
                        color * alpha_eff
                    )
                    opacity_accum[pixel.y, pixel.x] += alpha_eff * (1 - opacity_accum[pixel.y, pixel.x])

    return image

def backpropagate_gradient(loss, primitives, rendered_image, gt_image):
    """
    Backpropagates error through the rendering pipeline to update primitive features.
    """
    # Compute loss
    loss_val = compute_loss(rendered_image, gt_image)
    
    # Gradient w.r.t. image
    grad_img = compute_gradient(loss_val, rendered_image)
    
    # Backprop through compositing and rasterization
    # The Möller-Trumbore intersection is differentiable
    # Gradients flow to intersection points, then to primitive vertices/geometry
    for prim in primitives:
        # Update geometric features (position, distances, rotation)
        # Update SH coefficients
        prim.update_gradients(grad_img)
```

### Parameters
*   **SH Degree:** 48 coefficients (typically degree 2 or 3, stored as floats) [S1].
*   **Opacity Threshold:** 0.999 for stopping compositing [S1].
*   **Normalization Factor:** Smallest distance $d$ among all primitives [S1].

### Edge Cases and Pitfalls
*   **Scale Ambiguity:** Without normalization, appearance varies across different scene sizes. Handled by scaling distances relative to the smallest primitive distance $d$ [S1].
*   **Approximation Error:** The 3D affine ray space approximation error grows with distance from center points but remains negligible for small primitives [S1].
*   **Convexity:** The rasterization assumes primitives are convex, ensuring at most two intersections per ray [S1].

## Operating characteristics
*   **Time Complexity:** Real-time performance is achieved due to efficient parallelization in preprocessing and tiling-based rasterization [S1].
*   **Memory Complexity:** Dominated by the storage of 48 SH coefficients per primitive [S1]. Reducing SH degree or primitive count lowers memory usage [S1].
*   **Strengths:** Bounded, triangular geometry allows for better handling of sharp features compared to Gaussian splats; fully differentiable; real-time speeds [S1].
*   **Limitations:** Approximation error in ray space projection; scale ambiguity requires global normalization [S1].
*   **When to use:** Scenes requiring high-fidelity reconstruction of fine geometry and sharp edges where Gaussian splats may blur details [S1].

## Background
LinPrim was introduced in the paper "LinPrim: Linear Primitives for Differentiable Volumetric Rendering" [S1]. It builds on the 3D Gaussian Splatting pipeline but replaces Gaussian ellipsoids with transparent octahedron and tetrahedron primitives [S1]. The method was developed to provide an intuitive representation that can be easily understood and modified, while retaining real-time rendering speeds and comparable photometric reconstruction quality to existing methods [S1].

## Sources
[S1] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
