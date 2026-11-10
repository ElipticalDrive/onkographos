# Soft Rasterization

Also called: SAR Feature Rasterizer

## Overview
Soft Rasterization is a differentiable rendering technique that blends facet contributions for pixels near primitive edges to mitigate discontinuities, enabling the backpropagation of gradients through the rasterization stage [S1]. It is specifically adapted for Synthetic Aperture Radar (SAR) imaging geometry, which requires non-standard coordinate transformations (including ground-plane projection) distinct from standard optical rendering [S1]. Unlike hard rasterization, which produces sharp, non-differentiable edges, this method yields slightly distorted images that allow the learning of more useful gradients for downstream neural shaders [S1].

## How it works
The method implements a differentiable rasterizer based on the soft rasterization approach of [25], extended with a custom coordinate transformation sequence to handle SAR imaging geometry [S1]. The implementation extends the PyTorch3D API [S1].

### Inputs and Outputs
**Inputs:**
*   Mesh geometry (vertices and faces) in World Space.
*   Camera parameters (position, orientation).
*   Illumination direction $\hat{\mathbf{q}}$.
*   Blending radius $\sigma$ (controls distortion level).

**Outputs:**
*   Feature maps:
    1.  Silhouette mask (fragments with no corresponding facets).
    2.  Surface normal feature map (dot product of illumination and normal).
    3.  Alpha channel (visibility mask accounting for self-shadowing).

### Data Structures
The method relies on standard graphics primitives but modifies the rasterization kernel to output soft distances.
*   **Vertices:** 3D coordinates in World Space.
*   **Faces:** Indices referencing vertices.
*   **Coordinate Space:** The pipeline transforms coordinates through five spaces: World Space $\rightarrow$ View Space $\rightarrow$ Range Space $\rightarrow$ Ground Space $\rightarrow$ NDC Space [S1].

### Coordinate Transformations
The rasterizer applies a specific sequence of transforms to map World Space coordinates to NDC Space, accommodating SAR's slant-range and ground-plane geometry [S1]:

1.  **View Transform:** Transforms coordinates from World Space to View Space. Coordinates are relative to the camera’s point of view [S1].
2.  **Range Transform:** Transforms View Space to Range Space. The $z$-coordinate (depth/distance from camera) becomes the new $y$-coordinate [S1].
3.  **Ground-plane Projection:** Transforms Range Space to Ground Space. This stretches the perceived $y$-dimension such that each pixel maps to a consistent-sized cell on the ground, accounting for foreshortening/elongation/layover effects [S1]. Omitting this results in slant-plane geometry [S1].
4.  **Orthographic Projection:** Transforms Ground Space to NDC Space. Normalizes coordinate values into a range natively interpretable by the graphics processor. It skips perspective projection because distance from the camera does not affect perceived size in this SAR model [S1].

### Rasterization Steps
The rasterization consists of three stages: Coordinate Transformations, Rasterization, and Shading [S1].

1.  **Coordinate Transformation:** Apply the sequence defined above to all mesh vertices.
2.  **Soft Rasterization:** For each pixel, compute the contribution of nearby triangles. The soft rasterizer blends facet contributions for pixels near edges. This blending is controlled by the parameter $\sigma$ [S1].
    *   *Pitfall:* The blending yields slightly distorted images (artifacts shown in [S1] Figure 7), but this is necessary for differentiability [S1].
3.  **Shading Feature Map 1 (Silhouette):** Produce a mask out of fragments that have no corresponding facets. This identifies the silhouette of the object [S1].
4.  **Shading Feature Map 2 (Surface Normal):** Compute the surface normal feature value $d$ for each pixel:
    $$ d = -\hat{\mathbf{q}} \cdot \hat{\mathbf{n}} $$
    where:
    *   $d$ is the surface normal feature value.
    *   $\hat{\mathbf{q}}$ is the direction of illumination.
    *   $\hat{\mathbf{n}}$ is the per-pixel surface normal of the object.
    *   $\cdot$ denotes the inner-product [S1].
5.  **Shading Feature Map 3 (Alpha Channel):** Determine visibility to handle SAR self-shadowing:
    *   Rasterize a reference view from the perspective of the radar.
    *   Determine which facets are visible from this reference viewpoint.
    *   Mask out facets not visible from this reference viewpoint when calculating the alpha channel for the main view [S1].

### Parameters
*   **Blending radius $\sigma$:** Controls the level of soft rasterization distortion. Values used in training are $0$, $10^{-5}$, $10^{-4}$, and $10^{-3}$ [S1].
    *   *Note:* $\sigma=0$ corresponds to hard rasterization (non-differentiable edges). Non-zero values introduce the soft blending [S1].

### Pseudocode
The following pseudocode outlines the SAR Feature Rasterizer pipeline. It assumes standard vector operations and a soft rasterization kernel `soft_rasterize` which returns interpolated attributes based on $\sigma$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sar_feature_rasterizer(mesh, camera_params, illumination_direction, sigma):
    """
    Renders SAR feature maps using soft rasterization.
    
    Args:
        mesh: Object containing vertices (V) and faces (F) in World Space.
        camera_params: Camera position and orientation.
        illumination_direction: Unit vector q_hat.
        sigma: Blending radius for soft rasterization.
        
    Returns:
        silhouette_map: 2D mask of object silhouette.
        normal_map: 2D map of surface normal feature d.
        alpha_map: 2D visibility mask.
    """
    
    # 1. Coordinate Transformations
    # Transform vertices from World Space to NDC Space
    # Sequence: World -> View -> Range -> Ground -> NDC
    
    # View Transform: World to View Space
    view_vertices = apply_view_transform(mesh.vertices, camera_params)
    
    # Range Transform: View to Range Space (z becomes y)
    range_vertices = apply_range_transform(view_vertices)
    
    # Ground-plane Projection: Range to Ground Space
    ground_vertices = apply_ground_plane_projection(range_vertices)
    
    # Orthographic Projection: Ground to NDC Space
    ndc_vertices = apply_orthographic_projection(ground_vertices)
    
    # 2. Soft Rasterization
    # Rasterize the mesh in NDC space using soft rasterization
    # Returns interpolated attributes per pixel
    # Attributes needed: depth, barycentric coordinates, face indices
    raster_output = soft_rasterize(
        faces=mesh.faces,
        vertices=ndc_vertices,
        sigma=sigma,
        image_width=camera_params.width,
        image_height=camera_params.height
    )
    
    # 3. Shading Feature Map 1: Silhouette
    # Mask out fragments that have no corresponding facets
    # soft_rasterize typically returns a visibility weight or depth
    # If weight is 0 or depth is invalid, it's background
    silhouette_map = raster_output.visibility_weight > 0
    
    # 4. Shading Feature Map 2: Surface Normal Feature
    # Compute per-pixel surface normal n_hat
    # This requires interpolating face normals or computing from barycentrics
    per_pixel_normal = interpolate_normals(
        faces=mesh.faces,
        face_normals=compute_face_normals(mesh.vertices, mesh.faces),
        barycentrics=raster_output.barycentric_coordinates,
        vertices=ndc_vertices # Note: normals should be transformed appropriately
    )
    
    # Normalize the interpolated normal
    per_pixel_normal = normalize(per_pixel_normal)
    
    # Compute d = -q_hat . n_hat
    normal_map = -dot_product(illumination_direction, per_pixel_normal)
    
    # 5. Shading Feature Map 3: Alpha Channel (Visibility)
    # Handle SAR self-shadowing by rasterizing a reference view
    
    # Define reference camera at radar position
    radar_camera_params = get_radar_camera_params(camera_params)
    
    # Transform vertices for reference view (same sequence)
    ref_view_vertices = apply_view_transform(mesh.vertices, radar_camera_params)
    ref_range_vertices = apply_range_transform(ref_view_vertices)
    ref_ground_vertices = apply_ground_plane_projection(ref_range_vertices)
    ref_ndc_vertices = apply_orthographic_projection(ref_ground_vertices)
    
    # Rasterize reference view
    ref_raster_output = soft_rasterize(
        faces=mesh.faces,
        vertices=ref_ndc_vertices,
        sigma=sigma,
        image_width=camera_params.width,
        image_height=camera_params.height
    )
    
    # Determine visible facets from reference view
    # Create a mask of visible pixels in the reference view
    ref_visibility = ref_raster_output.visibility_weight > 0
    
    # Map reference visibility back to main view facets
    # Any facet not visible in reference view is masked out in main view
    # This is a simplification; typically involves depth comparison or face visibility flags
    # For implementation: if a pixel in main view corresponds to a face that is 
    # occluded in the reference view, set alpha to 0.
    
    # Simplified logic: Use reference depth to determine occlusion
    # If main view depth > ref view depth (or similar logic depending on coordinate system),
    # the facet is occluded.
    
    alpha_map = compute_alpha_with_self_shadowing(
        main_raster_output=raster_output,
        ref_raster_output=ref_raster_output,
        visibility_mask=ref_visibility
    )
    
    return silhouette_map, normal_map, alpha_map

def apply_view_transform(vertices, camera_params):
    # Standard view matrix multiplication
    return vertices @ camera_params.view_matrix

def apply_range_transform(vertices):
    # z becomes y, y becomes z (or similar swap depending on convention)
    # Based on [S1]: z-coordinates become new y-coordinates
    # Assuming input is (x, y, z), output is (x, z, y) or similar
    # Note: Exact mapping depends on coordinate system handedness.
    # We assume a swap of y and z for this description.
    x = vertices[:, 0]
    y = vertices[:, 2] # Old z becomes new y
    z = vertices[:, 1] # Old y becomes new z (or 0 for orthographic)
    return torch.stack([x, y, z], dim=1)

def apply_ground_plane_projection(vertices):
    # Stretches y-dimension
    # This is a scaling operation based on SAR geometry
    # Scale factor depends on range and ground resolution
    scale_factor = get_ground_scale_factor(vertices[:, 1]) # Based on new y (range)
    vertices[:, 1] *= scale_factor
    return vertices

def apply_orthographic_projection(vertices):
    # Normalize to NDC [-1, 1]
    # Assuming vertices are in a known range
    ndc_x = (vertices[:, 0] - min_x) / (max_x - min_x) * 2 - 1
    ndc_y = (vertices[:, 1] - min_y) / (max_y - min_y) * 2 - 1
    ndc_z = (vertices[:, 2] - min_z) / (max_z - min_z) * 2 - 1
    return torch.stack([ndc_x, ndc_y, ndc_z], dim=1)

def soft_rasterize(faces, vertices, sigma, image_width, image_height):
    # Placeholder for the soft rasterization kernel from [25]
    # Returns:
    # - visibility_weight: float tensor [H, W]
    # - barycentric_coordinates: tensor [H, W, 3]
    # - depth: tensor [H, W]
    # - face_indices: tensor [H, W]
    pass

def interpolate_normals(faces, face_normals, barycentrics, vertices):
    # Interpolate face normals using barycentric coordinates
    # Note: Face normals should be in the same space as the rasterized vertices
    # If vertices are in NDC, normals must be transformed accordingly
    # For orthographic projection, rotation is preserved, so normal rotation applies
    # This is a simplified interpolation
    return barycentrics @ face_normals

def compute_alpha_with_self_shadowing(main_raster_output, ref_raster_output, visibility_mask):
    # Logic to mask out facets occluded in reference view
    # This is complex and depends on specific implementation details not fully provided
    # General approach:
    # 1. Identify which faces are visible in ref view
    # 2. For each pixel in main view, check if its face is in the visible set
    # 3. If not, alpha = 0
    pass
```

## Operating characteristics
*   **Time Complexity:** Determined by the rasterization kernel and the number of faces. Soft rasterization is generally more expensive than hard rasterization due to the blending calculations, but still linear in the number of fragments processed [S1].
*   **Memory Complexity:** Requires storing intermediate feature maps (silhouette, normal, alpha) and potentially reference view buffers for self-shadowing calculation [S1].
*   **Strengths:** Enables gradient-based optimization of mesh geometry and camera parameters via backpropagation [S1]. Handles SAR-specific imaging effects like foreshortening and layover through coordinate transformations [S1].
*   **Limitations:** Produces visually distorted images due to edge blending (artifacts shown in [S1] Figure 7) [S1]. The self-shadowing calculation requires an additional rasterization pass [S1].
*   **When to use:** When differentiable rendering is required for SAR imagery generation or optimization [S1].
*   **When not to use:** When photorealistic, artifact-free rendering is the primary goal without the need for gradients [S1].

## Background
The Soft Rasterization method described here is part of the "SAR Feature Rasterizer" introduced in the paper "Differentiable Rendering for Synthetic Aperture Radar Imagery" [S1]. It builds on the soft rasterization technique from [25] (cited in [S1], likely referring to "Soft Rasterizer: A Differentiable Renderer for Image-based 3D Generative Adversarial Networks" by Laine et al., though the exact citation is not provided in the source text, the reference [25] is mentioned). The method was developed to adapt differentiable rendering for the unique geometry of SAR, which differs from standard optical imaging [S1]. It is used in the context of training a Neural Shader (CGAN) to predict realistic SAR shading effects [S1].

## Sources
[S1] Differentiable Rendering for Synthetic Aperture Radar Imagery, https://doi.org/10.48550/arxiv.2204.01248, fetched 2026-10-26
