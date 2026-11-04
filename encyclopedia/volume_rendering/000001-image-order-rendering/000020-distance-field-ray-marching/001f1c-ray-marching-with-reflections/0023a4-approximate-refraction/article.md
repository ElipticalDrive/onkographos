# Approximate Refraction

Also called: Normal-based texture offset refraction

## Overview
Approximate refraction is a technique to make a fluid surface appear to refract objects behind it, providing a visual effect similar to ray tracing but with significantly lower computational cost and implementation complexity [S1, unofficial]. It differs from [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md) by avoiding the physical simulation of light paths through the volume, instead relying on a 2D texture coordinate offset derived from the surface normal [S1, unofficial]. It is distinct from [Reflected Ray Directions](2028-reflected-ray-directions.md) as it targets transmission rather than reflection, and from [Cone Marching](3808-cone-marching.md) by using point-sampled normals rather than volume-aware cone intersections for the offset calculation.

## How it works
**Inputs:**
- A background scene rendered into a texture.
- A volume representing the fluid surface.
- View direction $V$ and up vector $z$.
- Scalar parameter $\beta > 0$.

**Outputs:**
- Final image with refracted appearance.

**Data Structures:**
1. **Background Texture**: Stores the color of objects behind the fluid.
2. **Hit Location Texture**: Stores the 2D screen-space coordinates $p$ of the nearest ray-surface intersection.
   - Alpha channel: $1$ if intersection exists, $0$ otherwise.
3. **Shading Normal Texture**: Stores the surface normal $N$ at the intersection point.

**Steps:**
1. **Render Background**: Render all scene geometry located behind the fluid volume into a background texture.
2. **Ray Marching**: For each pixel, march through the volume to find the nearest intersection with the fluid surface.
   - Store the screen-space hit location $p$ in the Hit Location Texture.
   - Store the shading normal $N$ at the hit point in the Shading Normal Texture.
   - Set the alpha of the Hit Location Texture to $0$ if no intersection is found, $1$ otherwise.
3. **Compute Basis Vectors**: Calculate the orthonormal basis vectors $P_h$ and $P_v$ for the image plane based on the view direction $V$ and up vector $z$.
4. **Calculate Texture Offset**: For each pixel with a valid hit (alpha $= 1$), compute the texture coordinate offset using the surface normal $N$ and parameter $\beta$.
5. **Sample Background**: Look up the color in the Background Texture using the calculated texture coordinate $t$.
6. **Composite**: Use the sampled background color in the shading equation for the fluid surface. Add foreground objects to create the final image.

**Formulas:**
The texture coordinate $t$ is calculated as:
$$t = p - \beta(N \cdot P_h, N \cdot P_v)$$

Where:
- $p$ is the location of the pixel being shaded (screen-space coordinates).
- $N$ is the surface normal at the hit point.
- $\beta$ is a scalar parameter controlling the severity of the refraction effect ($\beta > 0$) [S1, unofficial].
- $P_h$ and $P_v$ are orthonormal basis vectors for the image plane.

The vertical basis vector $P_v$ is calculated as:
$$P_v = \frac{z - (z \cdot V)V}{\|z - (z \cdot V)V\|}$$

Where:
- $z$ is the up vector.
- $V$ is the view direction.

The horizontal basis vector $P_h$ is calculated as:
$$P_h = P_v \times V$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_approximate_refraction(background_texture, hit_location_texture, normal_texture, view_direction, up_vector, beta):
    """
    Computes the approximate refraction effect for a fluid surface.
    
    Args:
        background_texture: Texture containing the scene behind the fluid.
        hit_location_texture: Texture containing screen-space hit locations (p) and validity (alpha).
        normal_texture: Texture containing surface normals (N) at hit locations.
        view_direction: View vector V.
        up_vector: Up vector z.
        beta: Scalar parameter controlling refraction severity.
        
    Returns:
        Final image texture.
    """
    # Precompute basis vectors for the entire frame
    # Project up vector onto plane perpendicular to view direction
    z_proj = up_vector - dot(up_vector, view_direction) * view_direction
    norm_z_proj = length(z_proj)
    
    # Handle degenerate case where up_vector is parallel to view_direction
    if norm_z_proj < 1e-6:
        # Fallback: use a default horizontal vector, e.g., (1, 0, 0)
        # This is a standard graphics fallback [G]
        P_v = normalize(cross(view_direction, (1, 0, 0)))
    else:
        P_v = z_proj / norm_z_proj
        
    P_h = cross(P_v, view_direction)
    
    final_image = empty_texture_like(hit_location_texture)
    
    for each pixel (x, y):
        # Get hit location and normal
        p = hit_location_texture[x, y].xy
        alpha = hit_location_texture[x, y].a
        N = normal_texture[x, y].xyz
        
        if alpha == 0:
            # No intersection, use background or default color
            final_image[x, y] = background_texture[x, y]
            continue
            
        # Calculate texture coordinate offset
        # N dot P_h gives the horizontal component of the normal in the image plane
        # N dot P_v gives the vertical component
        offset_x = N.x * P_h.x + N.y * P_h.y + N.z * P_h.z # Dot product N . P_h
        offset_y = N.x * P_v.x + N.y * P_v.y + N.z * P_v.z # Dot product N . P_v
        
        t_x = p.x - beta * offset_x
        t_y = p.y - beta * offset_y
        
        # Sample background texture at offset coordinate
        refracted_color = background_texture[t_x, t_y]
        
        # Use refracted color in shading (simplified here as direct color)
        # In a full shader, this would be combined with lighting models [G]
        final_image[x, y] = refracted_color
        
    return final_image
```

**Edge Cases and Pitfalls:**
- **Degenerate View Direction**: If the up vector $z$ is parallel to the view direction $V$, the denominator in the $P_v$ calculation becomes zero. A fallback basis vector must be chosen [G].
- **Boundary Conditions**: Texture coordinates $t$ may fall outside the $[0, 1]$ range. Standard texture wrapping or clamping modes should be applied when sampling the background texture.
- **Precision**: The offset magnitude depends on $\beta$ and the magnitude of the normal's projection. Large normals or large $\beta$ values can cause significant displacement, potentially sampling incorrect background regions.

## Subtopics

### [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md)
Physically based refraction with absorption. Choose when accurate light transport through the volume is required.

### [Reflected Ray Directions](2028-reflected-ray-directions.md)
Handles reflections. Choose when mirror-like surfaces are needed.

### [Cone Marching](3808-cone-marching.md)
Uses cone intersections for better volume accuracy. Choose when high-quality volume rendering is needed and performance allows.

## Operating characteristics
- **Time Complexity**: $O(1)$ per pixel for the refraction offset calculation, assuming constant-time texture lookups. The dominant cost is the ray marching step to find the surface intersection.
- **Memory Complexity**: Requires storage for the background texture, hit location texture, and normal texture.
- **Performance**: Significantly faster than ray tracing for refraction, as it avoids casting secondary rays into the scene [S1, unofficial].
- **Strengths**: Simple to implement, fast, suitable for real-time applications.
- **Limitations**: Does not account for the actual path of light through the volume; can produce artifacts if the background scene is complex or if the surface curvature is extreme. May not find intersections with other scene geometry as accurately as ray tracing [S1, unofficial].

## Background
The method was described in "Real-Time Simulation and Rendering of 3D Fluids" [S1, unofficial]. It was developed as a practical alternative to ray tracing for fluid animation in interactive environments, where performance is critical. The technique leverages the fact that for many fluid simulations, the visual impact of refraction can be approximated by distorting the background image based on the surface normal.

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
