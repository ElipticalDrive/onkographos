# Projective Space Texture Mapping with Multi-Resolution Templates

Also called: Inverse/Forward Projection Texture Indexing, Plenoptic Surface Texture Mapping, Projective Space Texture, Projective Texture Mapping for Volume, Pseudo-Solid Texturing

## Overview
Projective Space Texture Mapping with Multi-Resolution Templates is a technique that assigns solid textures to any position within a volume object using plenoptic projection functions as pseudo-procedure functional models [S1, unofficial]. It solves the problem of intermixing volumetric rendering with solid texturing without requiring explicit $uv$-parameterization of the volume geometry [S1, unofficial]. Unlike sibling methods that rely on fixed geometric parameterizations, this approach connects 3D positions in the volume to 2D projective positions on a plenoptic surface, and then to integral coordinates in a morphed texture image [S1, unofficial]. It is distinguished by its use of multi-resolution intermediate templates generated via forward projection (DVR/DSR) and inverse projection (projective texture mapping) to mitigate aliasing artifacts known as the "cliff effect" [S1, unofficial].

## How it works
The method operates in two distinct phases: forward projection to generate multi-resolution intermediate templates, and inverse projection to map textures from a morphed image onto the volume [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume object data.
    *   A morphed texture image (2D array of color/texture values).
    *   Plenoptic primitive centroid $(O_x, O_y, O_z)$.
    *   Resolution parameters for templates and patches.
*   **Outputs:**
    *   Textured volume rendering.
    *   Intermediate templates (images) at various resolutions.

### Data Structures
1.  **Morphed Texture Image:** A 2D grid of size $W \times H$. Coordinates $(u, v)$ are normalized continuous variables in $[0, 1]$.
2.  **Intermediate Template:** A 2D image representing the plenoptic surface. Indexed by integral coordinates $(x, y)$ corresponding to resolution intervals $\Delta u, \Delta v$.
3.  **High-Resolution Patch:** A sub-region of the morphed texture image defined by vertices $(u_{a1}, v_{a1})$ and $(u_{a2}, v_{a2})$.

### Formulas and Definitions
*   **Plenoptic Surface Parameterization:** Defined by two continuous variables $u \in [0, 1]$ and $v \in [0, 1]$ [S1, unofficial].
*   **Resolution Intervals (Forward Projection):**
    For an intermediate template with dimensions $x_d \times y_d$:
    $$ \Delta u = \frac{1}{x_d - 1} $$
    $$ \Delta v = \frac{1}{y_d - 1} $$
    A pixel at continuous position $(u, v)$ is indexed by integral pair $(x, y)$ where:
    $$ u = x \cdot \Delta u $$
    $$ v = y \cdot \Delta v $$
*   **High-Resolution Patch Indexing (Inverse Projection):**
    For a patch with vertices $(u_{a1}, v_{a1})$ and $(u_{a2}, v_{a2})$ where $u_{a1} < u_{a2}$ and $v_{a1} < v_{a2}$, the position $(x_a, y_a)$ in the patch for a texture coordinate $(u, v)$ is:
    $$ (x_a, y_a) = \left( \frac{u - u_{a1}}{\Delta u_a}, \frac{v - v_{a1}}{\Delta v_a} \right) $$
    where $\Delta u_a$ and $\Delta v_a$ are the resolution intervals (width/height) of the patch [S1, unofficial].
*   **Ray Origin Adjustment (Forward Projection):**
    For a new high-resolution representation of a region with positions $(u_1, v_1)$ and $(u_2, v_2)$, and new resolution intervals $\Delta u_1 = \frac{1}{x_{d1}-1}$ and $\Delta v_1 = \frac{1}{y_{d1}-1}$, the new origin of the tracing ray is:
    $$ Origin(u, v) = \left( u_1 + x_1 \cdot (u_2 - u_1) \cdot \Delta u_1, \quad v_1 + y_1 \cdot (v_2 - v_1) \cdot \Delta v_1 \right) $$
    where $0 \le x_1 \le x_{d1}-1$ and $0 \le y_1 \le y_{d1}-1$ [S1, unofficial].

### Numbered Steps

**Phase 1: Forward Projection (Template Generation)**
1.  Define the plenoptic surface parameterization using continuous variables $u \in [0, 1]$ and $v \in [0, 1]$ [S1, unofficial].
2.  Select a resolution $x_d \times y_d$ for the intermediate template. Calculate intervals $\Delta u = 1/(x_d - 1)$ and $\Delta v = 1/(y_d - 1)$ [S1, unofficial].
3.  Iterate through integral coordinates $(x, y)$ for the template. Calculate continuous positions $u = x \cdot \Delta u$ and $v = y \cdot \Delta v$ [S1, unofficial].
4.  Use $u$ and $v$ to adjust the origin and direction of the tracing ray for Direct Volume Rendering (DVR) or Direct Surface Rendering (DSR). The ray origin is adjusted according to the formula:
    $$ Origin(u, v) = \left( u_1 + x_1 \cdot (u_2 - u_1) \cdot \Delta u_1, \quad v_1 + y_1 \cdot (v_2 - v_1) \cdot \Delta v_1 \right) $$
    (Note: The source uses $x_1, y_1$ in the formula context of generating the plenoptic surface, implying these map to the template grid indices) [S1, unofficial].
5.  Render the ray to generate the pixel value for the intermediate template at $(x, y)$ [S1, unofficial].
6.  Repeat for desired resolutions to create a multi-resolution representation of the intermediate template [S1, unofficial].

**Phase 2: Inverse Projection (Volume Texturing)**
1.  For each voxel $p(X, Y, Z)$ in the volume object, define a ray passing through $p(X, Y, Z)$ and the plenoptic primitive centroid $(O_x, O_y, O_z)$ [S1, unofficial].
2.  Determine the 2D projective position $(u, v)$ on the plenoptic surface where this ray intersects or is parameterized [S1, unofficial].
3.  Identify the corresponding high-resolution patch in the morphed texture image that covers $(u, v)$. Let the patch vertices be $(u_{a1}, v_{a1})$ and $(u_{a2}, v_{a2})$ [S1, unofficial].
4.  Calculate the integral coordinates $(x_a, y_a)$ in the morphed texture image using:
    $$ (x_a, y_a) = \left( \frac{u - u_{a1}}{\Delta u_a}, \frac{v - v_{a1}}{\Delta v_a} \right) $$
    where $\Delta u_a, \Delta v_a$ are the patch resolution intervals [S1, unofficial].
5.  Sample the texture value from the morphed texture image at $(x_a, y_a)$ [S1, unofficial].
6.  Assign this texture value to the voxel $p(X, Y, Z)$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_intermediate_template(volume_data, centroid, width, height):
    """
    Generates an intermediate template using forward projection (DVR/DSR).
    """
    # Resolution intervals
    delta_u = 1.0 / (width - 1)
    delta_v = 1.0 / (height - 1)
    
    template = empty_image(width, height)
    
    for y in range(height):
        for x in range(width):
            # Continuous coordinates on plenoptic surface
            u = x * delta_u
            v = y * delta_v
            
            # Adjust ray origin for DVR/DSR based on u, v
            # Note: The source formula uses u1, v1, u2, v2, x1, y1.
            # Assuming u1,v1 and u2,v2 define the global bounds [0,1]x[0,1]
            # and x1,y1 correspond to the current grid indices x,y.
            u1, v1 = 0.0, 0.0
            u2, v2 = 1.0, 1.0
            
            # Calculate new origin for the tracing ray
            origin_x = u1 + x * (u2 - u1) * delta_u
            origin_y = v1 + y * (v2 - v1) * delta_v
            
            # Define ray direction towards centroid or based on projection geometry
            # [G] Specific ray direction calculation is not detailed in source,
            # assuming standard projective ray from origin through (u,v) plane.
            ray_origin = (origin_x, origin_y, 0.0) # Placeholder for 3D origin
            ray_direction = normalize((centroid.x - ray_origin.x, 
                                       centroid.y - ray_origin.y, 
                                       centroid.z - ray_origin.z))
            
            # Perform DVR/DSR
            color = render_ray(volume_data, ray_origin, ray_direction)
            
            template[y, x] = color
            
    return template

def texture_volume(volume_data, morphed_texture, centroid, template_resolution):
    """
    Assigns textures to voxels using inverse projection.
    """
    textured_volume = empty_volume(volume_data.shape)
    
    # Define a high-resolution patch in the morphed texture
    # [G] In practice, this might be the entire image or a specific region.
    # Assuming the whole morphed texture is the patch for simplicity.
    u_a1, v_a1 = 0.0, 0.0
    u_a2, v_a2 = 1.0, 1.0
    
    # Resolution intervals for the patch (assuming patch covers full texture)
    tex_width, tex_height = morphed_texture.shape
    delta_u_a = 1.0 / (tex_width - 1)
    delta_v_a = 1.0 / (tex_height - 1)
    
    for z in range(volume_data.shape[2]):
        for y in range(volume_data.shape[1]):
            for x in range(volume_data.shape[0]):
                voxel_pos = (x, y, z)
                
                # 1. Define ray through voxel and centroid
                ray_origin = voxel_pos
                ray_direction = normalize((centroid.x - ray_origin.x, 
                                           centroid.y - ray_origin.y, 
                                           centroid.z - ray_origin.z))
                
                # 2. Determine 2D projective position (u, v)
                # [G] The source states p(X,Y,Z) is indexed by a ray passing through
                # p and centroid. It connects 3D position to 2D projective position.
                # The exact mapping from 3D ray to 2D (u,v) on the plenoptic surface
                # depends on the specific plenoptic primitive definition.
                # Assuming (u,v) are derived from the ray's intersection with a 
                # reference plane or parameterization.
                # [G] Since the source does not give the explicit formula for 
                # (u,v) from the ray, we assume a standard projective mapping 
                # where (u,v) are normalized coordinates relative to the centroid 
                # and viewing frustum.
                u, v = get_projective_coordinates(ray_origin, ray_direction, centroid)
                
                # 3. Map (u,v) to texture coordinates
                # Ensure u, v are within patch bounds
                if not (u_a1 <= u <= u_a2 and v_a1 <= v <= v_a2):
                    continue
                    
                # 4. Calculate integral coordinates in morphed texture
                tex_x = (u - u_a1) / delta_u_a
                tex_y = (v - v_a1) / delta_v_a
                
                # 5. Sample texture
                # [G] Bilinear interpolation should be used for continuous (u,v)
                texture_value = sample_texture(morphed_texture, tex_x, tex_y)
                
                # 6. Assign to voxel
                textured_volume[z, y, x] = texture_value
                
    return textured_volume

def get_projective_coordinates(ray_origin, ray_direction, centroid):
    """
    [G] Placeholder for the specific geometric projection function.
    The source describes the connection between 3D position and 2D 
    projective position but does not provide the explicit algebraic 
    formula for this transformation step. It relies on the plenoptic 
    primitive definition.
    """
    # Standard projective mapping example:
    # Assuming a pinhole camera model at centroid looking at a plane
    # This is an implementation gap filled with standard graphics knowledge.
    pass
```

### Parameters and Typical Values
*   **Morphed Texture Image Size:** Experiments show that $128 \times 128$ pixels results in prominent cliff effects, while $1024 \times 1024$ pixels greatly improves image quality [S1, unofficial].
*   **Resolution Intervals:** Determined by the dimensions of the intermediate template or patch ($x_d, y_d$) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Cliff Effect:** A single pixel projected onto a surface almost parallel to the direction of projection is smeared over a relatively large area of solid, introducing alias artifacts [S1, unofficial].
*   **Mitigation:** Use high-resolution texture images and high-resolution intermediate templates to eliminate these artifacts reasonably [S1, unofficial].
*   **Degenerate Input:** If the ray direction is parallel to the surface normal or the projection plane, the $(u, v)$ coordinates may become undefined or infinite. The source does not specify handling for this, but standard projective geometry techniques (e.g., clipping) would be required [G].

## Subtopics

### Direct Volume Rendering (DVR) / Direct Surface Rendering (DSR)
Used in the forward projection phase to generate intermediate templates. Sets itself apart by rendering the volume/surface from adjusted ray origins defined by the plenoptic parameters [S1, unofficial].

### Projective Texture Mapping
Used in the inverse projection phase. Sets itself apart by mapping the morphed texture directly to volume voxels via ray indexing, avoiding explicit mesh parameterization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels and the cost of DVR/DSR for template generation. Inverse projection requires ray tracing for each voxel to determine $(u, v)$ and texture sampling.
*   **Memory Complexity:** Requires storage for the volume data, the morphed texture image, and potentially multiple intermediate templates if multi-resolution is used [S1, unofficial].
*   **Strengths:** Avoids degradation of rendered image quality while interpolating texture to resolutions exceeding the texture map. Eliminates the need for $uv$-parameterization [S1, unofficial].
*   **Limitations:** Finite resolution of the morphed texture image can still introduce alias artifacts (cliff effect) if not sufficiently high [S1, unofficial].
*   **When to Use:** When texturing complex volumetric scenes without explicit surface parameterization is required, and high-quality solid texturing is needed [S1, unofficial].
*   **When Not to Use:** When computational resources are limited and high-resolution textures cannot be afforded, leading to visible aliasing [S1, unofficial].

## Background
The method was introduced in the paper "Texture Mapping Volume Objects" [S1, unofficial]. It builds on the concept of plenoptic functions and solid textures, addressing the intermixing of volumetric rendering and solid texturing. It references earlier work on two-part texture mapping by Bier and Sloan [ES86] and solid texture models by Winter [Win02]. The approach uses plenoptic indexing functions to substitute mathematical/procedure models for projective space texturing [S1, unofficial].

## Sources
[S1] Texture Mapping Volume Objects, https://doi.org/10.2312/vvg.20051006, fetched 2026-10-30 (unofficial)
