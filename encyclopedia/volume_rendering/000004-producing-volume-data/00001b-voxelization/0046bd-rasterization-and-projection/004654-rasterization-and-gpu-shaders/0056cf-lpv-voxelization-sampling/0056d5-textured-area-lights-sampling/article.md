# Textured Area Lights Sampling

Also called: Mipmapped Radiant Flux Sampling

## Overview
Textured Area Lights Sampling is a method to store and sample radiant flux information on area light surfaces using texture mapping and mipmapping to approximate the entire radiant flux of an intersection area [S2, unofficial]. It solves the problem of efficiently voxelizing area light source meshes into Light Propagation Volumes (LPV) by creating Virtual Point Lights (VPLs) at intersection areas, simulating direct illumination more efficiently than dense sampling [Parent]. It sets itself apart from siblings by using a pre-computed radiant flux map sampled via mipmaps to approximate the flux of the projected area, rather than dense point sampling or direction-based AABB emission sampling [Siblings].

## How it works
**Inputs:**
*   Triangle mesh representing the area light source.
*   Radiant flux map: A texture storing radiant flux information corresponding to the triangle mesh surface [S1, unofficial].
*   LPV grid structure.
*   Intersection area: The projection of the light triangle onto a specific LPV face [S1, unofficial].

**Outputs:**
*   Radiant flux value for the intersection area, used to populate LPV cells [S2, unofficial].

**Data Structures:**
*   **Radiant Flux Map:** A 2D texture where texels store radiant flux values. The mapping from triangle surface to texture is defined by vertex texture coordinates [S1, unofficial].
*   **AABB:** Axis-Aligned Bounding Box of the intersection area projection on the LPV face [S1, unofficial].

**Steps:**
1.  **Compute Intersection AABB:** Determine the Axis-Aligned Bounding Box (AABB) of the intersection area projection on the LPV face where the triangle is projected [S1, unofficial][S2, unofficial]. Note that this intersection area can cover two LPV cells; for correct radiance distribution, the intersection area for every intersected LPV cell must be processed separately [S1, unofficial][S2, unofficial].
2.  **Compute AABB Center Texture Coordinate:**
    *   Calculate the center of the AABB in the texture space or barycentric space of the triangle.
    *   Compute the barycentric coordinates of the AABB center relative to the triangle vertices [S1, unofficial][S2, unofficial].
    *   Interpolate the texture coordinates at the triangle vertices using these barycentric coordinates to obtain the texture coordinate $(u_c, v_c)$ at the AABB center [S1, unofficial][S2, unofficial].
3.  **Compute Mipmap Level:**
    *   Determine the required mipmap level to approximate the radiant flux of the intersection area.
    *   Calculate the mipmap level based on the texture coordinates in the corners of the AABB, following OpenGL standard specifications [S1, unofficial][S2, unofficial].
    *   The OpenGL standard specification for mipmap level calculation (Segal and Akeley, 1999) typically involves computing the maximum gradient of the texture coordinates across the primitive. For an AABB in texture space with corners $(u_1, v_1), (u_2, v_2), (u_3, v_3), (u_4, v_4)$, the level $L$ is derived from the maximum partial derivatives $\max(|\frac{\partial u}{\partial x}|, |\frac{\partial u}{\partial y}|, |\frac{\partial v}{\partial x}|, |\frac{\partial v}{\partial y}|)$ [S1, unofficial][S2, unofficial]. *Note: The exact formula from Segal and Akeley is not explicitly provided in the source text, but the method relies on it.*
4.  **Sample Radiant Flux:**
    *   Sample the radiant flux map at $(u_c, v_c)$ using the computed mipmap level. This returns the approximate total radiant flux for the intersection area [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_textured_area_light(triangle, radiant_flux_map, lpv_face):
    """
    Samples the radiant flux for an intersection area of a triangle on an LPV face.
    
    Args:
        triangle: Object with vertices v0, v1, v2.
                  Each vertex has position p and texture coordinate uv.
        radiant_flux_map: Texture object with sample_mipmap(u, v, level) method.
        lpv_face: The LPV face geometry for projection.
        
    Returns:
        float: Approximated radiant flux for the intersection area.
    """
    # Step 1: Compute AABB of intersection area projection on LPV face
    # This requires projecting the triangle onto the LPV face and finding the intersection polygon,
    # then computing its AABB. The source notes state this is similar to the non-homogeneous radiance method.
    intersection_polygon = compute_intersection_polygon(triangle, lpv_face)
    aabb = compute_aabb(intersection_polygon)
    
    # Step 2: Compute texture coordinate at AABB center
    # Calculate barycentric coordinates of the AABB center relative to the triangle
    center_pos = aabb.center()
    barycentric = compute_barycentric_coordinates(center_pos, triangle.v0, triangle.v1, triangle.v2)
    
    # Interpolate texture coordinates
    u_c = barycentric.w0 * triangle.v0.uv.x + barycentric.w1 * triangle.v1.uv.x + barycentric.w2 * triangle.v2.uv.x
    v_c = barycentric.w0 * triangle.v0.uv.y + barycentric.w1 * triangle.v1.uv.y + barycentric.w2 * triangle.v2.uv.y
    
    # Step 3: Compute mipmap level based on AABB corners
    # Get texture coordinates for all four corners of the AABB in texture space
    # Note: The AABB is in LPV face space, so we need to project its corners to the triangle's texture space
    # or compute the AABB directly in texture space if the projection is affine.
    # Assuming we have the texture coordinates of the AABB corners in texture space:
    aabb_corners_uv = [
        compute_texture_coord_for_aabb_corner(aabb, corner_index=0, triangle=triangle),
        compute_texture_coord_for_aabb_corner(aabb, corner_index=1, triangle=triangle),
        compute_texture_coord_for_aabb_corner(aabb, corner_index=2, triangle=triangle),
        compute_texture_coord_for_aabb_corner(aabb, corner_index=3, triangle=triangle)
    ]
    
    # Compute mipmap level following OpenGL standard (Segal and Akeley, 1999)
    # This typically involves calculating the maximum gradient of u and v across the primitive.
    # For an axis-aligned box in texture space, the gradients are simply the width and height.
    u_min = min(c.u for c in aabb_corners_uv)
    u_max = max(c.u for c in aabb_corners_uv)
    v_min = min(c.v for c in aabb_corners_uv)
    v_max = max(c.v for c in aabb_corners_uv)
    
    # Approximate gradients (assuming unit pixel size or normalized texture space)
    # The exact OpenGL formula is complex, but generally:
    # level = log2(max(max_grad_u, max_grad_v))
    # Here, we approximate the gradients as the dimensions of the AABB in texture space.
    # [G] The source does not provide the exact OpenGL formula implementation details.
    # A common approximation for the mipmap level L for a rectangle of size (du, dv) in normalized texture space:
    du = u_max - u_min
    dv = v_max - v_min
    max_grad = max(du, dv)
    
    # Handle degenerate case where AABB has zero size
    if max_grad < 1e-6:
        level = 0
    else:
        # [G] Standard OpenGL mipmap level calculation involves the maximum partial derivative.
        # For a simple AABB in texture space, the level is often approximated by log2 of the max dimension.
        # This is a simplification; the exact Segal and Akeley method should be used for precision.
        level = log2(max_grad)
        
    # Clamp level to valid range
    level = clamp(level, 0, radiant_flux_map.max_level())
    
    # Step 4: Sample the radiant flux map
    flux = radiant_flux_map.sample_mipmap(u_c, v_c, level)
    
    return flux
```

**Parameters:**
*   **Mipmap Level:** Computed dynamically based on the size of the intersection area in texture space. No fixed typical value is given; it scales with the projected area size [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Z-axis Intersection:** Computing a single radiant flux sample for all LPV cells intersected along the z-axis can cause minor visual errors [S1, unofficial][S2, unofficial]. The intersection area must be processed for *every* intersected LPV cell separately if it covers multiple cells along the projection axis [S1, unofficial][S2, unofficial].
*   **Mipmap Approximation Error:** Due to the approximation made by the mipmapping technique, radiant flux information can be introduced from areas outside the intersection area [S1, unofficial][S2, unofficial]. This is a known limitation of using mipmaps for flux approximation.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per intersection area, assuming constant-time texture sampling and mipmap level calculation. The cost is dominated by the intersection computation and texture lookup.
*   **Memory Complexity:** Depends on the size of the radiant flux map. The map must store flux information for the entire light source mesh [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with the number of light triangles, as each triangle is processed independently. Performance may degrade if the radiant flux map is very large or if mipmap generation is expensive.
*   **Strengths:** Efficient approximation of radiant flux for area lights using standard texture sampling hardware. Simpler than dense sampling [S1, unofficial].
*   **Limitations:** Potential visual errors due to z-axis intersection handling and flux leakage from mipmapping [S1, unofficial][S2, unofficial]. Accuracy depends on the quality of the radiant flux map and the mipmap approximation.
*   **When to use:** When efficient direct illumination approximation is needed for area lights in LPV, and high precision is not critical. Suitable for real-time applications where dense sampling is too expensive.
*   **When not to use:** When high-fidelity direct illumination is required, or when the light source has complex geometry that cannot be well-approximated by a 2D texture map.

## Background
Textured Area Lights Sampling was introduced in the context of Light Propagation Volumes (LPV) for real-time global illumination. The method was proposed to address the lack of a strategy in the original LPV technique to sample the surface of area light sources efficiently [S1, unofficial]. It builds on the concept of voxelizing light triangles into the LPV grid and using texture mapping to store and retrieve flux information. The approach references the OpenGL standard specifications for mipmap level calculation (Segal and Akeley, 1999) [S1, unofficial][S2, unofficial].

## Sources
[S1] , https://www.scitepress.org/publishedPapers/2022/108416/pdf/index.html, fetched 2026-10-29 (unofficial)
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2022/108416/, fetched 2026-10-29 (unofficial)
