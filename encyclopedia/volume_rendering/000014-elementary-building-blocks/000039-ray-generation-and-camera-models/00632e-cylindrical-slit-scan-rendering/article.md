# Cylindrical Slit-Scan Rendering

Also called: Cylindrical Panorama Rendering, Slit-Scan Rendering

## Overview
Cylindrical Slit-Scan Rendering generates a cylindrical environment map by performing a series of narrow vertical rendering passes, each capturing a single column of the final panorama. This method eliminates horizontal perspective distortion by ensuring the camera's optical axis remains perpendicular to the projection plane for every column [S1]. It is distinct from standard panoramic cameras or single-pass equirectangular projections by decoupling the horizontal view angle from the vertical field of view, allowing for precise control over the vertical frustum while maintaining a consistent horizon [S1].

## How it works
**Inputs and Outputs**
- **Input**: A 3D scene (e.g., Gaussian Splatting or mesh-based scene), vertical field of view ($FoV_Y$), target width $w$, and target height $h$.
- **Output**: A cylindrical perspective map of dimensions $w \times h$.

**Data Structures**
- **Cylindrical Map**: A 2D image buffer of size $w \times h$.
- **Rendering Strip**: A temporary buffer of size $16 \times h$ (or similar narrow width) used for intermediate rasterization.

**Parameters**
- **Vertical Field of View ($FoV_Y$)**: Fixed at $120^\circ$ [S1].
- **Strip Width**: 16 pixels [S1].
- **Up Vector**: $[0, 1, 0]$ [S1].

**Steps**
1. **Scene Alignment**: Align the scene's vertical axis to the up-vector $[0, 1, 0]$ [S1].
2. **Camera Positioning**: Position a virtual camera at the center of the scene (origin $\mathbf{o} = [0, 0, 0]$) [S1].
3. **Iterative Rendering**: For each column index $i$ from $0$ to $w-1$:
   a. Calculate the horizontal angle $\theta_i$:
      $$ \theta_i = \frac{2\pi \cdot i}{w} + \pi $$
   b. Set the camera view direction $\mathbf{d}$:
      $$ \mathbf{d} = [\sin(\theta_i), 0, \cos(\theta_i)] $$
   c. Set the camera up vector to $[0, 1, 0]$ to maintain a consistent horizontal horizon [S1].
   d. Render a narrow vertical strip of width 16 pixels centered on the column corresponding to $i$, using the fixed $FoV_Y$ of $120^\circ$ [S1].
   e. Extract the central column of pixels from the rendered strip.
4. **Assembly**: Concatenate the extracted columns in order of $i$ to form the final $w \times h$ Cylindrical Perspective Map [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cylindrical_slit_scan_render(scene, w, h, fov_y_degrees=120):
    """
    Generates a cylindrical panorama using slit-scan rendering.
    
    Args:
        scene: The 3D scene data structure.
        w: Width of the output cylindrical map.
        h: Height of the output cylindrical map.
        fov_y_degrees: Vertical field of view in degrees (default 120).
        
    Returns:
        cylindrical_map: A 2D array of shape (w, h) containing the rendered pixels.
    """
    # Initialize output buffer
    cylindrical_map = empty_2d_array(w, h)
    
    # Camera setup
    camera_origin = [0, 0, 0]
    up_vector = [0, 1, 0]
    fov_y_rad = radians(fov_y_degrees)
    
    # Render each column
    for i in range(w):
        # Calculate horizontal angle
        theta_i = (2 * PI * i / w) + PI
        
        # Calculate view direction
        view_direction = [sin(theta_i), 0, cos(theta_i)]
        
        # Configure camera for this pass
        # Note: The specific rasterizer API is abstracted here.
        # The camera is positioned at origin, looking at view_direction,
        # with up_vector [0,1,0] and vertical FOV fov_y_rad.
        camera = Camera(
            origin=camera_origin,
            direction=view_direction,
            up=up_vector,
            fov_y=fov_y_rad,
            aspect_ratio=1.0 # Assumed for narrow strip processing
        )
        
        # Render a narrow strip (e.g., 16 pixels wide)
        # The strip is centered around the column i
        strip_width = 16
        strip_x_start = i - strip_width // 2
        strip_x_end = i + strip_width // 2
        
        # Render the strip to a temporary buffer
        # The rasterizer handles the projection and shading
        strip_buffer = render_scene(scene, camera, 
                                   x_range=[strip_x_start, strip_x_end], 
                                   y_range=[0, h])
        
        # Extract the central column from the strip
        # Assuming strip_buffer has width 'strip_width'
        center_col_index = strip_width // 2
        central_column = strip_buffer[:, center_col_index]
        
        # Place the column in the final map
        cylindrical_map[i, :] = central_column
        
    return cylindrical_map
```

**Edge Cases and Pitfalls**
- **Horizontal Boundaries**: The method naturally handles the wrap-around from $i=w-1$ to $i=0$ due to the periodic nature of the sine/cosine functions, but care must be taken if the scene is not centered or if the camera origin is not truly central [S1].
- **Vertical FoV Limitation**: The fixed $120^\circ$ vertical field of view means that polar regions (zenith and nadir) are not covered by the cylindrical map, resulting in empty areas [S1].

## Subtopics

### Equirectangular Projection and Inpainting
This variant extends the cylindrical map to a full spherical equirectangular projection. It involves mapping cylindrical coordinates to equirectangular coordinates and filling the empty polar regions using generative inpainting. For each pixel $(x', y')$ in the target equirectangular image, the source vertical coordinate $v_{source}$ in the cylindrical map is calculated as $v_{source} = \tan(\phi) / \tan(FoV_Y / 2)$, where $\phi$ is the vertical angle corresponding to $y'$ and $FoV_Y$ is $120^\circ$ [S1]. The polar regions, which are empty due to the limited vertical FoV, are filled using generative inpainting (e.g., Stable Diffusion via Krita Generative AI plugin) with prompts such as "realistic sky" [S1]. To avoid seams at horizontal boundaries, an offset is applied and a second inpainting pass is performed [S1]. Additionally, regions containing extra elongated Gaussians in the foreground (which may cover the bottom part of the map) are painted black to be filled by the inpainting process [S1].

## Operating characteristics
- **Time Complexity**: $O(w \cdot h \cdot r)$, where $r$ is the cost of rendering a single narrow strip. Since $w$ passes are performed, the total time is proportional to the width of the panorama.
- **Memory Complexity**: $O(w \cdot h)$ for the final map, plus $O(16 \cdot h)$ for the temporary strip buffer.
- **Strengths**: Eliminates horizontal perspective distortion, which is critical for accurate environment mapping in certain applications [S1]. Simple to implement using standard rasterizers.
- **Limitations**: Does not cover the full sphere (polar regions are missing) [S1]. Requires multiple rendering passes, which can be slower than a single-pass method for very wide panoramas.
- **When to Use**: When horizontal perspective accuracy is paramount and the scene is centered. Suitable for generating environment maps for Gaussian Splatting or other novel view synthesis techniques [S1].
- **When Not to Use**: When full spherical coverage is required without post-processing, or when computational efficiency is critical and horizontal distortion is acceptable.

## Background
The method was introduced by D. Pintani et al. in the paper "EnvMap-GS: two-stage outdoor Gaussian reconstruction with background-to-environment map baking" [S1]. It was developed to facilitate the integration of cylindrical environment maps into existing renderers, particularly for outdoor Gaussian reconstruction tasks where background elements like sky and distant buildings need to be accurately represented [S1]. The approach leverages the fact that cylindrical projections eliminate horizontal perspective distortion, making them suitable for certain visualization and reconstruction pipelines [S1].

## Sources
[S1] EnvMap-GS: two-stage outdoor Gaussian reconstruction with background-to-environment map baking, https://doi.org/10.1007/s00371-026-04593-w, fetched 2026-10-29
