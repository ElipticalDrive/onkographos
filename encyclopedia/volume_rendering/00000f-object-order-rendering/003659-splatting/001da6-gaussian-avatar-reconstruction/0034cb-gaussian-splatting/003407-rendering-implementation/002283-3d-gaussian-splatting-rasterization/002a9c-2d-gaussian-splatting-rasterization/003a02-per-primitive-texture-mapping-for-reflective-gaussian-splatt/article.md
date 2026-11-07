# Per-Primitive Texture Mapping for Reflective Gaussian Splatting

Also called: RefGaussian, TextureSplat

## Overview
Per-Primitive Texture Mapping for Reflective Gaussian Splatting (TextureSplat) enhances 2D Gaussian Splatting by replacing single attribute values per primitive with spatially varying texture maps for material properties and normals [S1]. Unlike methods that assign constant attributes to each primitive, this approach decouples geometric representation from appearance, enabling high-frequency specular reconstruction and accurate reflections in reflective scenes without increasing the primitive count [S1]. It builds on [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md) by leveraging the closed-form ray-splat intersection to map screen pixels to local texture coordinates, allowing hardware-accelerated rendering via texture atlases [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D Gaussian primitives (surfels), each defined by position, tangential vectors, scaling factors, and opacity. Each primitive $k$ has an associated texture map $T^k$ containing material properties (e.g., albedo, roughness, normal maps) in its local coordinate system.
*   **Output:** A rendered image with accurate specular highlights and reflections.

**Data Structures:**
*   **Primitive $k$:**
    *   $p_k$: Position in world space.
    *   $t_{uk}, t_{vk}$: Tangential vectors in world space.
    *   $s_{uk}, s_{vk}$: Scaling factors along tangential directions.
    *   $o_k$: Learnable opacity scalar.
    *   $T^k$: Texture map (e.g., RGBA for color/alpha, RGB for normals) stored in a unified texture atlas.
*   **Texture Atlas:** A single large texture containing all primitive textures $T^k$, packed to leverage GPU hardware-accelerated texture filtering [S1].

**Numbered Steps:**

1.  **Primitive Definition:** Represent the scene using oriented planar disks (2DGS primitives). For each primitive $k$, define the position $p_k$, tangential vectors $t_{uk}$ and $t_{vk}$, and scaling factors $s_{uk}, s_{vk}$ [S1]. The primitive's normal is computed as $n_k = t_{uk} \times t_{vk}$ [S1].

2.  **Ray-Splat Intersection:** For each pixel $(x, y)$, determine the intersection with contributing primitives. Using the closed-form ray-splat intersection of 2DGS, map screen-space pixels to local texture coordinates $(u_k, v_k)$ for each primitive $k$ [S1]. The local coordinates correspond to the point on the primitive's tangent plane [S1].

3.  **Gaussian Evaluation:** Evaluate the 2D Gaussian value $G_k(u, v)$ at the local coordinates $(u_k, v_k)$:
    $$ G_k(u, v) = \exp \left( - \frac{u^2 + v^2}{2} \right) $$
    The effective opacity $\alpha_k(x, y)$ for primitive $k$ at pixel $(x, y)$ is:
    $$ \alpha_k(x, y) = o_k G_k(u_k(x, y), v_k(x, y)) $$
    [S1]

4.  **Attribute Retrieval:** Retrieve material attributes $a_k(x, y)$ from the per-primitive texture map $T^k$ using the local texture coordinates:
    $$ a_k(x, y) = T^k(u_k(x, y), v_k(x, y)) $$
    [S1]
    If the attribute is a normal map, the retrieved normal is in the tangent space of the primitive.

5.  **Normal Transformation:** Transform tangential normal maps to world space. For each primitive, construct a rotation matrix $R_k$ from the tangential vectors $t_{uk}$ and $t_{vk}$ (and the normal $n_k$). Transform the sampled tangent-space normal $n_{tangent}$ to world space:
    $$ n_{world} = R_k \cdot n_{tangent} $$
    [S1]

6.  **Physically-Based Rendering (PBR):** Use the retrieved material attributes (e.g., albedo, roughness, world-space normal) in a physically-based rendering model compatible with deferred shading pipelines to compute the final color contribution, including specular reflections [S1].

7.  **Alpha Compositing:** Accumulate the color contributions of all primitives contributing to pixel $(x, y)$ in front-to-back order. The final pixel color $A(x, y)$ is:
    $$ A(x, y) = \sum_{i=1}^{N} a_i \alpha_i(x, y) \prod_{j=1}^{i-1} (1 - \alpha_j(x, y)) $$
    where $a_i$ is the color attribute retrieved from $T^i$ (or computed via PBR), and $\alpha_i(x, y)$ is the effective opacity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def evaluate_2d_gaussian(u, v):
    """Evaluate the 2D Gaussian function at local coordinates (u, v)."""
    return math.exp(-(u**2 + v**2) / 2.0)

def transform_normal_to_world(tangent_u, tangent_v, normal, tangent_normal):
    """
    Transform a tangent-space normal to world space using the primitive's basis.
    tangent_u, tangent_v, normal: World space basis vectors (orthonormal).
    tangent_normal: Normal vector in tangent space (RGB).
    Returns: World space normal vector (RGB).
    """
    # Construct rotation matrix from basis vectors
    # R = [tangent_u, tangent_v, normal]
    # n_world = R * n_tangent
    n_world = (
        tangent_u * tangent_normal[0] +
        tangent_v * tangent_normal[1] +
        normal * tangent_normal[2]
    )
    return n_world

def render_pixel(pixels, primitives, texture_atlas, atlas_uv_offsets, atlas_uv_scales):
    """
    Render a single pixel using per-primitive texture mapping.
    
    Args:
        pixels: Array of pixel coordinates (x, y) to render.
        primitives: List of primitive objects. Each primitive has:
            - p_k: Position (3D vector)
            - t_uk, t_vk: Tangential vectors (3D vectors)
            - s_uk, s_vk: Scaling factors (scalars)
            - o_k: Opacity (scalar)
            - texture_id: Index into texture atlas
        texture_atlas: 2D array representing the texture atlas.
        atlas_uv_offsets: Mapping from primitive_id to (u_offset, v_offset) in atlas.
        atlas_uv_scales: Mapping from primitive_id to (u_scale, v_scale) in atlas.
        
    Returns:
        Rendered image array.
    """
    # Note: Actual ray-splat intersection logic is complex and depends on camera parameters.
    # This pseudocode assumes we have already computed local coordinates (u_k, v_k)
    # for each primitive k that intersects the pixel.
    
    # Initialize output
    image = np.zeros((pixels.shape[0], 3))
    
    for pixel_idx, (x, y) in enumerate(pixels):
        # Collect all primitives contributing to this pixel
        # In practice, this involves sorting primitives by depth for front-to-back blending
        contributing_primitives = []
        
        for k, prim in enumerate(primitives):
            # Step 1 & 2: Ray-splat intersection to get local coordinates (u_k, v_k)
            # This is a placeholder for the closed-form intersection logic from 2DGS
            # u_k, v_k = compute_ray_splat_intersection(prim, camera, x, y)
            # For the sake of this pseudocode, assume u_k, v_k are computed
            u_k, v_k = 0.0, 0.0 # Placeholder
            
            # Step 3: Evaluate Gaussian
            g_k = evaluate_2d_gaussian(u_k, v_k)
            alpha_k = prim.o_k * g_k
            
            if alpha_k > 1e-6:
                contributing_primitives.append((k, alpha_k, u_k, v_k))
        
        # Sort primitives by depth (front-to-back)
        # Note: Depth calculation depends on camera and primitive position
        contributing_primitives.sort(key=lambda item: item[0]) # Placeholder for depth sort
        
        accumulated_alpha = 0.0
        accumulated_color = np.zeros(3)
        
        for k, alpha_k, u_k, v_k in contributing_primitives:
            # Step 4: Retrieve attributes from texture atlas
            # Map local (u_k, v_k) to atlas UV coordinates
            prim = primitives[k]
            u_atlas = (u_k * prim.s_uk) * atlas_uv_scales[k][0] + atlas_uv_offsets[k][0]
            v_atlas = (v_k * prim.s_vk) * atlas_uv_scales[k][1] + atlas_uv_offsets[k][1]
            
            # Sample texture atlas
            # Note: Texture filtering (bilinear/trilinear) is handled by GPU hardware
            # Here we assume a simple sample function
            texture_sample = sample_texture_atlas(texture_atlas, u_atlas, v_atlas)
            
            # texture_sample contains albedo, roughness, normal, etc.
            albedo = texture_sample[:3]
            roughness = texture_sample[3]
            tangent_normal = texture_sample[4:7]
            
            # Step 5: Transform normal to world space
            world_normal = transform_normal_to_world(
                prim.t_uk, prim.t_vk, 
                np.cross(prim.t_uk, prim.t_vk), 
                tangent_normal
            )
            
            # Step 6: PBR shading
            # Compute color based on albedo, normal, roughness, and lighting
            # This is a placeholder for the PBR model
            color = compute_pbr_color(albedo, world_normal, roughness, prim.p_k)
            
            # Step 7: Alpha compositing
            weight = alpha_k * (1.0 - accumulated_alpha)
            accumulated_color += color * weight
            accumulated_alpha += weight
            
            if accumulated_alpha > 0.99:
                break
        
        image[pixel_idx] = accumulated_color
        
    return image

def sample_texture_atlas(atlas, u, v):
    """
    Sample the texture atlas at normalized coordinates (u, v).
    Assumes atlas is a numpy array and uses bilinear filtering.
    """
    # Placeholder for actual texture sampling logic
    # In a real implementation, this would use GPU texture sampling
    return np.zeros(7) # Placeholder return
```

**Parameters:**
*   **Texture Atlas Size:** Not given in the sources.
*   **Filtering:** Hardware-accelerated texture filtering (bilinear/trilinear) is used [S1].
*   **PBR Model:** Compatible with deferred shading pipelines [S1]. Specific model details are not given.

**Edge Cases and Pitfalls:**
*   **Precision:** Local coordinates $(u, v)$ must be computed accurately to ensure correct texture sampling [S1].
*   **Boundaries:** Texture coordinates may fall outside the primitive's local bounds; texture filtering handles this, but clipping may be needed [S1].
*   **Degenerate Input:** If tangential vectors are collinear, the normal is undefined. This is a general 2DGS issue [S1].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of primitives per pixel, similar to standard 2DGS, but with additional texture sampling and PBR shading costs [S1].
*   **Memory Complexity:** Memory usage increases due to the storage of texture maps for each primitive, packed into a texture atlas [S1].
*   **Measured Performance:** Achieves real-time rendering performance on standard benchmarks [S1].
*   **Scaling:** Scales with the number of primitives and texture resolution [S1].
*   **Strengths:** High-frequency specular reconstruction, accurate reflections, decoupling of geometry and appearance, hardware acceleration [S1].
*   **Limitations:** Increased memory usage for textures, complexity of texture atlas management [S1].
*   **When to Use:** Reflective scenes, scenes with high-frequency material variations [S1].
*   **When Not to Use:** Scenes with simple, uniform materials where single attribute values suffice [S1].

## Background
*   **Origin:** Introduced by Mae Younes and Adnane Boukhayma from Inria France in the paper "TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting" (arXiv:2506.13348v2, Dec 2025) [S1].
*   **History:** Builds on 2D Gaussian Splatting (2DGS) [18] and 3D Gaussian Splatting [27] [S1].
*   **Usage:** Novel view synthesis, reflective scene reconstruction [S1].

## Sources
[S1] TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting, https://hal.science/hal-05484246v1, fetched 2026-10-26
