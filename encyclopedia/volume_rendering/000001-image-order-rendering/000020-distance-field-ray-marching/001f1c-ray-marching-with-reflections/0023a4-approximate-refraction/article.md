# Approximate Refraction
## Overview
Approximate refraction is a technique to make a fluid surface appear to refract objects behind it, which is faster and simpler to implement than ray tracing [S1, unofficial]. Convex regions of the surface magnify the image behind them, concave regions shrink it, and flat regions allow rays to pass straight through [S1, unofficial].

## How it works
**Inputs:**
- A background scene rendered into a texture [S1, unofficial].
- A volume representing the fluid surface [S1, unofficial].
- View direction $V$ and up vector $z$ [S1, unofficial].
- Scalar parameter $\beta > 0$ [S1, unofficial].

**Outputs:**
- Final image with refracted appearance [S1, unofficial].

**Data Structures:**
1. **Background Texture**: Stores the color of objects behind the fluid [S1, unofficial].
2. **Hit Location Texture**: Stores the 2D screen-space coordinates $p$ of the nearest ray-surface intersection. Alpha channel is $1$ if an intersection exists, $0$ otherwise [S1, unofficial].
3. **Shading Normal Texture**: Stores the surface normal $N$ at the intersection point [S1, unofficial].

**Steps:**
1. **Render Background**: Render all scene geometry located behind the fluid volume into a background texture [S1, unofficial].
2. **Ray Marching**: For each pixel, march through the volume to find the nearest intersection with the fluid surface [S1, unofficial].
   - Store the screen-space hit location $p$ in the Hit Location Texture [S1, unofficial].
   - Store the shading normal $N$ at the hit point in the Shading Normal Texture [S1, unofficial].
   - Set the alpha of the Hit Location Texture to $0$ if no intersection is found, $1$ otherwise [S1, unofficial].
3. **Compute Basis Vectors**: Calculate the orthonormal basis vectors $P_h$ and $P_v$ for the image plane based on the view direction $V$ and up vector $z$ [S1, unofficial].
4. **Calculate Texture Offset**: For each pixel with a valid hit (alpha $= 1$), compute the texture coordinate offset using the surface normal $N$ and parameter $\beta$ [S1, unofficial].
5. **Sample Background**: Look up the color in the Background Texture using the calculated texture coordinate $t$ [S1, unofficial].
6. **Composite**: Use the sampled background color in the shading equation for the fluid surface. Add foreground objects to create the final image [S1, unofficial].

**Formulas:**
The texture coordinate $t$ is calculated as:
$$t = p - \beta(N \cdot P_h, N \cdot P_v)$$

Where:
- $p$ is the location of the pixel being shaded (screen-space coordinates) [S1, unofficial].
- $N$ is the surface normal at the hit point [S1, unofficial].
- $\beta$ is a scalar parameter controlling the severity of the refraction effect ($\beta > 0$) [S1, unofficial].
- $P_h$ and $P_v$ are orthonormal basis vectors for the image plane [S1, unofficial].

The vertical basis vector $P_v$ is calculated as:
$$P_v = \frac{z - (z \cdot V)V}{\|z - (z \cdot V)V\|}$$

Where:
- $z$ is the up vector [S1, unofficial].
- $V$ is the view direction [S1, unofficial].

The horizontal basis vector $P_h$ is calculated as:
$$P_h = P_v \times V$$
[S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_approximate_refraction(background_texture, hit_location_texture, normal_texture, view_direction, up_vector, beta):
    # Precompute basis vectors for the image plane [G]
    z_proj = up_vector - (dot(up_vector, view_direction) * view_direction)
    norm_z_proj = length(z_proj)
    
    if norm_z_proj < 1e-6:
        # Fallback when up_vector is parallel to view_direction [G]
        P_v = normalize(cross(view_direction, (1.0, 0.0, 0.0)))
    else:
        P_v = z_proj / norm_z_proj
        
    P_h = cross(P_v, view_direction)
    
    final_image = create_empty_texture_like(hit_location_texture)
    
    for y in range(hit_location_texture.height):
        for x in range(hit_location_texture.width):
            p_xy = hit_location_texture[x, y].xy
            alpha = hit_location_texture[x, y].a
            N = normal_texture[x, y].xyz
            
            if alpha == 0:
                final_image[x, y] = background_texture[x, y]
                continue
                
            offset_x = dot(N, P_h)
            offset_y = dot(N, P_v)
            
            t_x = p_xy.x - beta * offset_x
            t_y = p_xy.y - beta * offset_y
            
            # Standard texture sampling [G]
            refracted_color = background_texture.sample(t_x, t_y)
            final_image[x, y] = refracted_color
            
    return final_image
```

**Edge Cases and Pitfalls:**
- **Degenerate View Direction**: If the up vector $z$ is parallel to the view direction $V$, the denominator in the $P_v$ calculation becomes zero. A fallback basis vector must be chosen [G].
- **Boundary Conditions**: Texture coordinates $t$ may fall outside the $[0, 1]$ range. Standard texture wrapping or clamping modes should be applied when sampling the background texture [G].
- **Precision**: The offset magnitude depends on $\beta$ and the magnitude of the normal's projection. Large normals or large $\beta$ values can cause significant displacement, potentially sampling incorrect background regions [G].

## Subtopics

### [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md)
Physically based refraction with absorption. Choose when accurate light transport through the volume is required.

### [Reflected Ray Directions](2028-reflected-ray-directions.md)
Handles reflections. Choose when mirror-like surfaces are needed.

### [Cone Marching](3808-cone-marching.md)
Uses cone intersections for better volume accuracy. Choose when high-quality volume rendering is needed and performance allows.

## Operating characteristics
- **Time Complexity**: $O(1)$ per pixel for the refraction offset calculation, assuming constant-time texture lookups [G]. The dominant cost is the ray marching step to find the surface intersection [S1, unofficial].
- **Memory Complexity**: Requires storage for the background texture, hit location texture, and normal texture [G].
- **Performance**: Significantly faster than ray tracing for refraction, as it avoids casting secondary rays into the scene [S1, unofficial].
- **Strengths**: Simple to implement, fast, suitable for real-time applications [S1, unofficial].
- **Limitations**: Does not account for the actual path of light through the volume; can produce artifacts if the background scene is complex or if the surface curvature is extreme [S1, unofficial]. Ray tracing is an alternative but is expensive and may fail to find intersections with other scene geometry [S1, unofficial].

## Background
The method was described in "Real-Time Simulation and Rendering of 3D Fluids" [S1, unofficial]. It was developed as a practical alternative to ray tracing for fluid animation in interactive environments, where performance is critical [S1, unofficial]. The technique leverages the fact that for many fluid simulations, the visual impact of refraction can be approximated by distorting the background image based on the surface normal [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
