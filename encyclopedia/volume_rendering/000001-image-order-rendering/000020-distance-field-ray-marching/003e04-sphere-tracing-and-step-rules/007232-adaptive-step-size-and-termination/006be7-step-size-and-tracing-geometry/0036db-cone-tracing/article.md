# Cone Tracing
## Overview
Cone tracing is a volume rendering technique that approximates indirect lighting (diffuse and specular) and soft shadows by casting expanding cones into a voxel grid, accumulating color and opacity along the ray direction until full occlusion or a maximum distance is reached [S2, unofficial]. It distinguishes itself from standard ray casting by using a cone aperture to aggregate light from a volume of space, providing visually plausible soft shadow borders and indirect illumination [S2, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** Surface position ($pos$), surface normal ($N$), light direction ($L$), voxel grid texture (3D texture with mipmaps), voxel size ($voxelSize$), voxel count ($u\_voxelCount$), cone aperture parameter ($apertureTan2$), and a quality step multiplier ($quality$) [S2, unofficial].
*   **Output:** Accumulated color ($color.rgb$) and opacity ($color.a$) [S2, unofficial].

**Data Structures:**
*   **Voxel Texture:** A 3D texture where each texel stores color ($rgb$) and opacity ($a$) [S2, unofficial]. Mipmaps are pre-generated to handle different cone diameters at varying distances [S2, unofficial].
*   **Accumulator:** A `vec4` variable initialized to $(0, 0, 0, 0)$ [S2, unofficial].

**Parameters:**
*   $apertureTan2$: Defined as $2 \cdot \tan(\frac{apertureAngle}{2})$. A typical value corresponds to an aperture angle of 30 degrees, yielding $apertureTan2 \approx 0.536$ [S2, unofficial].
*   $quality$: A step size multiplier. A typical value is $0.7$ [S2, unofficial].
*   $voxelSize$: The physical size of a single voxel in the grid [S2, unofficial].

**Algorithm:**
The method traces a cone from the surface point in the direction of the light source (or hemisphere for indirect lighting) [S3, unofficial]. At each step, the cone's diameter is calculated based on the distance traveled and the aperture [S2, unofficial]. The algorithm samples the voxel texture at the current position, using a Level of Detail (LOD) determined by the cone's diameter to ensure appropriate filtering [S2, unofficial]. Color is accumulated using alpha blending, scaled by the unoccluded area [S2, unofficial].

1.  Initialize $color = (0, 0, 0, 0)$ and $dist = 0.0$ [S2, unofficial].
2.  Calculate effective squared aperture $a2 = \max(voxelSize, apertureTan2)$ to ensure a non-zero minimum aperture [S2, unofficial].
3.  Loop while $color.a < 1.0$ and $dist$ is within bounds:
    a.  Calculate current cone diameter: $diameter = \max(voxelSize, dist \cdot a2)$ [S2, unofficial].
    b.  Calculate LOD level: $lod = \log_2(diameter \cdot u\_voxelCount)$, clamped to a maximum of 10 [S2, unofficial].
    c.  Update sampling position: $pos = from + dir \cdot dist$ [S2, unofficial].
    d.  If $pos$ is outside the voxel grid bounds, break the loop [S2, unofficial].
    e.  Fetch voxel color $cc$ from the texture at $pos$ using $lod$ [S2, unofficial].
    f.  Accumulate color: $color += (1 - color.a) \cdot cc$ [S2, unofficial]. This scales the fetched color by the unoccluded area.
    g.  Advance distance: $dist += diameter \cdot quality$ [S2, unofficial].
4.  Return $color.rgb$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_trace(from_pos, dir, voxel_texture, voxel_size, voxel_count, aperture_tan2, quality=0.7, max_dist=10.0, grid_min=(0.0, 0.0, 0.0), grid_max=(1.0, 1.0, 1.0)):
    """
    Cone tracing algorithm for approximating indirect lighting and soft shadows.
    Args:
        from_pos: Surface position (vec3)
        dir: Light direction or hemisphere sample direction (vec3)
        voxel_texture: 3D texture with mipmaps storing (rgb, a)
        voxel_size: Physical size of a voxel
        voxel_count: Number of voxels along one axis
        aperture_tan2: 2 * tan(aperture_angle / 2)
        quality: Step size multiplier (default 0.7)
        max_dist: Maximum marching distance to prevent infinite loops
        grid_min, grid_max: Bounding box of the voxel grid
    Returns:
        color.rgb: Accumulated color
    """
    color = (0.0, 0.0, 0.0, 0.0)
    dist = 0.0
    # Ensure non-zero aperture
    a2 = max(voxel_size, aperture_tan2)
    
    while color[3] < 1.0 and dist < max_dist:
        # Calculate cone diameter at current distance
        diameter = max(voxel_size, dist * a2)
        
        # Calculate LOD based on diameter and grid resolution
        lod = (diameter * voxel_count).log2() if hasattr(diameter * voxel_count, 'log2') else math.log2(diameter * voxel_count)
        lod = max(0.0, min(10.0, lod))
        
        # Update position
        pos = from_pos + dir * dist
        
        # Check bounds (assumes normalized or world-space coordinates matching grid_min/max)
        if pos.x < grid_min[0] or pos.x > grid_max[0] or \
           pos.y < grid_min[1] or pos.y > grid_max[1] or \
           pos.z < grid_min[2] or pos.z > grid_max[2]:
            break
            
        # Fetch color from voxel texture with mipmapping
        cc = texture(voxel_texture, pos, lod)
        
        # Accumulate color with alpha blending
        # Scale by unoccluded area (1 - current_alpha)
        color = (
            color[0] + (1.0 - color[3]) * cc[0],
            color[1] + (1.0 - color[3]) * cc[1],
            color[2] + (1.0 - color[3]) * cc[2],
            color[3] + (1.0 - color[3]) * cc[3]
        )
        
        # Advance distance by a fraction of the diameter
        dist += diameter * quality
        
    return (color[0], color[1], color[2])
```

**Edge Cases and Pitfalls:**
*   **Geometric Interpretation:** The shape traced by linearly increasing radius is technically a cone with a hemisphere on the end (a swept sphere of linearly increasing radius). Intersection occurs if the distance to the nearest surface is less than the current test radius at any point along the horizontal center line [S1, unofficial].
*   **Light Leaking:** The method is computationally expensive and tricky to implement without artifacts such as light leaking, especially in complex geometry [S3, unofficial].
*   **Precision:** The LOD calculation must be clamped to prevent sampling errors at very large distances or very small diameters [S2, unofficial].
*   **Initialization:** Some implementations initialize $dist$ to $voxelSize \cdot offset$ to skip the immediate surface, though $dist = 0.0$ is standard for direct cone casting [G].
## Subtopics

### [Soft Shadows via Cone Tracing](6043-soft-shadows-via-cone-tracing.md)
Approximates soft shadow boundaries by aggregating extinction values within the cone. Distinguished by its specific application to light transport in participating media, where the cone aperture controls softness and scattering range.

### [Customized Cone Tracing](3447-customized-cone-tracing.md)
Variant with specific optimizations for custom hardware or scene requirements.

### [Voxel Cone Tracing for Ambient Occlusion](3544-voxel-cone-tracing-for-ambient-occlusion.md)
Specialized for computing ambient occlusion using cone traces.

### [Deferred Voxel Shading](4685-deferred-voxel-shading.md)
Integrates cone tracing into a deferred rendering pipeline.

### [Voxel Shading via Convolution](4778-voxel-shading-via-convolution.md)
Uses convolution techniques for shading.

### [Voxel Cone Tracing Global Illumination](6037-voxel-cone-tracing-global-illumination.md)
Extends cone tracing for full global illumination.

### [Anisotropic Voxel Representation and Filtering](11895-anisotropic-voxel-representation-and-filtering.md)
Uses anisotropic voxels for better filtering.

### [Voxel Cone Tracing with Temporal Multi-bounce](12745-voxel-cone-tracing-with-temporal-multi-bounce.md)
Incorporates temporal accumulation for multi-bounce lighting.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps in the cone trace [S2, unofficial]. The number of steps depends on the scene depth and the cone aperture [S2, unofficial]. Each step involves a texture fetch with LOD calculation [S2, unofficial].
*   **Memory Complexity:** Depends on the voxel grid size and the number of mipmaps [S2, unofficial]. Typically, a 3D texture with multiple mip levels is required [S2, unofficial].
*   **Performance:** Computationally expensive compared to simple ray casting due to the volume of samples and texture lookups [S3, unofficial].
*   **Strengths:** Provides visually plausible soft shadows and indirect lighting with reasonable performance [S2, unofficial]. Effective for real-time applications where full global illumination is too expensive [S2, unofficial].
*   **Limitations:** Prone to artifacts like light leaking if not implemented carefully [S3, unofficial]. Performance degrades with complex geometry or large voxel grids [S3, unofficial]. Not energy conservative without additional attenuation factors [S3, unofficial].
*   **When to Use:** Real-time rendering of scenes with participating media or where soft shadows and indirect lighting are desired [S2, unofficial].
*   **When Not to Use:** When high precision is required, or when the scene is static and offline rendering is acceptable (use path tracing) [S3, unofficial]. Also not suitable for scenes with extremely fine details that exceed the voxel resolution [S3, unofficial].
## Background
Cone tracing was introduced as an extension of sphere tracing for Signed Distance Field (SDF) rendering [S1, unofficial]. It was described in a thesis that discussed using cones to provide useful intersection information with minimal additional cost [S1, unofficial]. The technique has been described in the `gidemo` project report and codebase to approximate indirect lighting and soft shadows in voxel-based environments [S2, unofficial][S3, unofficial].
## Sources
[S1] Reading thesis on cone tracing, did the author make a mistake?, https://computergraphics.stackexchange.com/questions/9508/reading-thesis-on-cone-tracing-did-the-author-make-a-mistake, fetched 2026-10-26 (unofficial)
[S2] righier/gidemo, https://github.com/righier/gidemo, fetched 2026-10-26 (unofficial)
[S3] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
