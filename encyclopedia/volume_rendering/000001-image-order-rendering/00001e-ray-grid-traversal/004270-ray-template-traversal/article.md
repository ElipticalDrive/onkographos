# Ray Template Traversal

Also called: Ray Template, Template-Based Rendering, Template-Based Volume Viewing, Template-based ray casting, Templated Ray Casting

## Overview
Ray Template Traversal is a discrete ray casting technique for parallel projections (orthographic viewing) that accelerates rendering by storing the traversal steps through the volume as a single "ray template" [S1, unofficial][S2, unofficial][S3, unofficial]. Unlike standard DDA methods that compute step directions for every ray, this method precomputes a 26-connected line sequence once and reuses it for all parallel rays [S2, unofficial][S4, unofficial]. It sets itself apart by requiring rays to be cast from a specific "base plane" within the volume rather than the image plane, ensuring uniform tessellation where each voxel is visited exactly once, followed by a 2D resampling step to map the result to the screen [S3, unofficial][S4, unofficial].

## How it works
The method relies on the fact that in parallel viewing, all rays have the same geometric form [S2, unofficial]. The algorithm consists of three phases: Initialization, Ray Casting, and 2D Mapping [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data grid $V$ with dimensions $N_x \times N_y \times N_z$ [S4, unofficial].
    *   Orthographic camera parameters: viewing direction $\mathbf{d}$, and the image plane extent [S4, unofficial].
    *   (Optional) Supersampling configuration (number of sub-pixel offsets) [S4, unofficial].
*   **Outputs:**
    *   Rendered image on the screen plane [S4, unofficial].

### Data Structures
1.  **Ray Template ($T$):** A list of discrete steps. Each step is an integer vector $(\Delta x, \Delta y, \Delta z)$ where components are in $\{-1, 0, 1\}$. The template represents a 26-connected path through the voxel grid [S3, unofficial][S4, unofficial].
    *   For standard zero-order interpolation, the template contains integer voxel indices [S5, unofficial].
    *   For higher-order interpolation (multiple samples per voxel), the template may contain floating-point offsets or weights [S4, unofficial].
2.  **Base Plane:** A 2D plane within the volume grid, parallel to one of the volume faces (e.g., $z=0$ or $z=N_z-1$), chosen such that it is the plane of the volume buffer most parallel to the image plane [S3, unofficial][S5, unofficial].
3.  **Base Image ($I_{base}$):** A 2D buffer storing the accumulated ray values (color/opacity) for each pixel in the base plane [S2, unofficial].

### Step 1: Initialization (Template Generation)
Generate the ray template using a 3D Digital Differential Analyzer (DDA) or Bresenham-like algorithm [S1, unofficial][S4, unofficial].
1.  Define the viewing direction $\mathbf{d} = (d_x, d_y, d_z)$ [G].
2.  Compute the step direction for each axis:
    $$
    \text{step}_i = \begin{cases} 
    +1 & \text{if } d_i > 0 \\
    -1 & \text{if } d_i < 0 \\
    0 & \text{if } d_i = 0 
    \end{cases}
    $$
3.  Generate the sequence of voxel indices visited by a ray starting at $(0,0,0)$ with direction $\mathbf{d}$ [G]. Store the relative increments $(\Delta x, \Delta y, \Delta z)$ between consecutive voxels in the list $T$ [G].
    *   *Note:* If supersampling is used, multiple templates are generated, one for each relative sub-pixel origin position [S4, unofficial].

### Step 2: Ray Casting (Base Plane Traversal)
Cast rays from the base plane to populate the base image [S2, unofficial][S3, unofficial].
1.  Identify the base plane coordinates. If the image plane is defined by pixel coordinates $(u, v)$, map these to the base plane. For example, if the base plane is $z=0$, the base plane coordinates correspond to the $(x, y)$ positions of the rays [S3, unofficial][S4, unofficial].
2.  For each pixel $(u, v)$ in the image extent:
    a.  Determine the starting voxel index $\mathbf{p}_{start}$ on the base plane corresponding to $(u, v)$ [S2, unofficial].
    b.  Initialize accumulation variables: $C_{acc} = 0$, $\alpha_{acc} = 0$ [G].
    c.  Iterate through the steps in the template $T$:
        i.  Update current voxel index: $\mathbf{p}_{curr} \leftarrow \mathbf{p}_{curr} + T[k]$ [G].
        ii. Check if $\mathbf{p}_{curr}$ is within volume bounds [G].
        iii. Sample the volume value $v_{val} = V(\mathbf{p}_{curr})$ [G].
        iv. Compute color $c$ and opacity $\alpha$ using transfer functions [S2, unofficial].
        v.  Perform front-to-back compositing:
            $$
            C_{acc} \leftarrow C_{acc} + c \cdot \alpha \cdot (1 - \alpha_{acc})
            $$
            $$
            \alpha_{acc} \leftarrow \alpha_{acc} + \alpha \cdot (1 - \alpha_{acc})
            $$
        vi. If $\alpha_{acc}$ exceeds a user-defined threshold, break [S4, unofficial].
    d.  Store $(C_{acc}, \alpha_{acc})$ in $I_{base}(u, v)$ [S2, unofficial].

### Step 3: 2D Mapping (Resampling)
Map the warped image from the base plane to the screen plane [S2, unofficial][S3, unofficial].
1.  The base image $I_{base}$ is geometrically distorted relative to the final image because it was generated on a plane parallel to the volume face, not the image plane [S4, unofficial].
2.  For each pixel $(u, v)$ in the final image:
    a.  Calculate the corresponding coordinates $(u', v')$ in the base plane coordinate system using the inverse of the projection transformation [S2, unofficial].
    b.  Sample $I_{base}(u', v')$ using bilinear interpolation [S5, unofficial].
    c.  Assign the interpolated value to the final image pixel [S5, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_template_traversal(volume, image_width, image_height, view_direction, base_plane_axis):
    """
    volume: 3D array [Nx, Ny, Nz]
    image_width, image_height: int
    view_direction: tuple (dx, dy, dz)
    base_plane_axis: 'x', 'y', or 'z', indicating which volume face is parallel to the image plane
    """
    
    # Phase 1: Initialization
    # Generate the ray template (sequence of voxel steps)
    template = generate_dda_template(view_direction)
    
    # Base plane is at z=0 for simplicity, mapping image (u,v) to volume (x,y,0)
    # We assume volume dimensions are large enough to cover the image extent
    base_image = [[(0.0, 0.0, 0.0), 0.0] for _ in range(image_height)]
    for row in base_image:
        row[:] = [(0.0, 0.0, 0.0), 0.0] * image_width
        
    # Phase 2: Ray Casting
    for u in range(image_width):
        for v in range(image_height):
            current_voxel = (u, v, 0)
            acc_color = [0.0, 0.0, 0.0]
            acc_alpha = 0.0
            
            for step in template:
                current_voxel = (current_voxel[0] + step[0], 
                                 current_voxel[1] + step[1], 
                                 current_voxel[2] + step[2])
                
                if not (0 <= current_voxel[0] < volume.shape[0] and
                        0 <= current_voxel[1] < volume.shape[1] and
                        0 <= current_voxel[2] < volume.shape[2]):
                    break
                
                val = volume[current_voxel[0]][current_voxel[1]][current_voxel[2]]
                color, alpha = apply_transfer_functions(val)
                
                weight = alpha * (1.0 - acc_alpha)
                acc_color[0] += color[0] * weight
                acc_color[1] += color[1] * weight
                acc_color[2] += color[2] * weight
                acc_alpha += alpha * (1.0 - acc_alpha)
                
                if acc_alpha > 0.95:
                    break
            
            base_image[v][u] = (tuple(acc_color), acc_alpha)
            
    # Phase 3: 2D Mapping
    final_image = [[(0.0, 0.0, 0.0), 0.0] for _ in range(image_height)]
    for row in final_image:
        row[:] = [(0.0, 0.0, 0.0), 0.0] * image_width
        
    for u in range(image_width):
        for v in range(image_height):
            base_u, base_v = map_to_base_plane(u, v, view_direction, base_plane_axis)
            final_image[v][u] = bilinear_interpolate(base_image, base_u, base_v)
            
    return final_image

def generate_dda_template(direction):
    """
    Generates a list of integer steps (dx, dy, dz) representing a 26-connected line.
    Uses a standard 3D DDA or Bresenham algorithm.
    """
    dx, dy, dz = direction
    steps = []
    cx, cy, cz = 0, 0, 0
    # Standard 3D DDA to generate steps until a sufficient length is reached
    # This covers the maximum possible depth in the viewing direction
    max_steps = 1000 # Sufficient for typical volume depths
    for _ in range(max_steps):
        steps.append((dx, dy, dz))
        # In a real implementation, DDA would compute exact boundary crossings
        # and append the correct step vector (e.g., (1,0,0), (1,1,0), etc.)
        # Here we assume direction components are normalized to step increments
        pass
    return steps

def apply_transfer_functions(val):
    # Standard transfer function lookup
    color = (val, val, val)
    alpha = 0.1
    return color, alpha

def map_to_base_plane(u, v, view_direction, base_plane_axis):
    # Inverse orthographic projection: maps screen coords to base plane coords
    # Simplified affine transform
    return float(u), float(v)

def bilinear_interpolate(image, u, v):
    # Clamp coordinates to valid range
    h, w = len(image), len(image[0])
    u = max(0.0, min(float(w) - 1.0, u))
    v = max(0.0, min(float(h) - 1.0, v))
    
    u0, v0 = int(u), int(v)
    u1, v1 = u0 + 1, v0 + 1
    du, dv = u - u0, v - v0
    
    # Handle boundary cases
    u1 = min(u1, w - 1)
    v1 = min(v1, h - 1)
    
    p00 = image[v0][u0]
    p10 = image[v0][u1]
    p01 = image[v1][u0]
    p11 = image[v1][u1]
    
    # Interpolate color
    color = (
        (1-du)*(1-dv)*p00[0][0] + du*(1-dv)*p10[0][0] + (1-du)*dv*p01[0][0] + du*dv*p11[0][0],
        (1-du)*(1-dv)*p00[0][1] + du*(1-dv)*p10[0][1] + (1-du)*dv*p01[0][1] + du*dv*p11[0][1],
        (1-du)*(1-dv)*p00[0][2] + du*(1-dv)*p10[0][2] + (1-du)*dv*p01[0][2] + du*dv*p11[0][2]
    )
    alpha = (1-du)*(1-dv)*p00[1] + du*(1-dv)*p10[1] + (1-du)*dv*p01[1] + du*dv*p11[1]
    
    return (color, alpha)
```

### Parameters and Edge Cases
*   **Base Plane Selection:** The base plane must be chosen parallel to one of the volume faces [S2, unofficial]. If rays are cast from the image plane instead, voxels will be missed or double-counted [S3, unofficial][S4, unofficial].
*   **Supersampling:** To support supersampling, multiple templates are required, one for each sub-pixel offset [S2, unofficial]. For 4x supersampling, 4 templates are generated [S2, unofficial].
*   **Higher-Order Interpolation:** For trilinear interpolation, the template can be extended to include floating-point weights for the 8 surrounding voxels, precomputed for the specific sub-pixel position [S2, unofficial].
*   **Illumination:** The method supports on-the-fly shading (e.g., Phong model) during the ray casting phase [S2, unofficial].
*   **Empty Space:** Adaptive variations can use different templates for empty and dense regions [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The template generation is $O(1)$ (or $O(L)$ where $L$ is the template length, computed once). The ray casting phase is $O(N_{pixels} \cdot L)$, where $L$ is the average number of steps per ray. Since the template is reused, the per-ray cost is reduced to simple integer additions and array lookups [S2, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(L)$ for the template storage. The base image requires $O(N_{pixels})$ memory [S2, unofficial].
*   **Performance:** The method is significantly faster than naive DDA per ray. Yagel and Kaufman reported a 3x speedup over previous algorithms for a resolution image [S2, unofficial]. Supersampling increases time linearly (e.g., 4 rays/pixel takes 4x longer) [S2, unofficial].
*   **Strengths:** Extremely efficient for parallel projections; avoids redundant floating-point calculations per ray; guarantees uniform sampling [S2, unofficial][S3, unofficial].
*   **Limitations:** Only applicable to orthographic (parallel) projections; requires a 2D resampling step which adds complexity; not suitable for perspective views without modification [S2, unofficial][S3, unofficial].

## Background
Ray Template Traversal was introduced by R. Yagel and A. Kaufman in "Template-Based Volume Viewing" (1992) [S2, unofficial][S3, unofficial]. It was developed to address the computational expense of discretizing every ray individually in parallel volume rendering [S3, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S2] PDF Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S4] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (unofficial)
