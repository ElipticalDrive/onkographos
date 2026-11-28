# Multi-view Lenticular Volume Rendering

Also called: GPU-accelerated volume rendering to 3D lenticular displays

## Overview
A method for hardware-accelerated generation of multi-view stereoscopic images of volumetric datasets for display on a multi-view lenticular screen, using the GPU for both rendering and compositing [S1, unofficial]. It solves the problem of interactive visualization of large 3D medical volumes on glasses-free 3D displays by decomposing the scene into perspective projections from offset focal spots and interlacing them into a single bitmap [S1, unofficial]. It sets itself apart from sibling methods by explicitly handling the asymmetric frustum geometry required for lenticular lenses and performing the final sub-pixel interlacing via a static texture map lookup on the GPU, rather than pre-computing a hologram or projecting from separate physical sources [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (e.g., medical CT/MRI), lenticular display parameters (focal distance $f$, view count $N$, camera spacing $d$), and the dataset dimensions [S1, unofficial].
*   **Output:** A single interlaced bitmap suitable for the LCD panel underlying the lenticular sheet [S1, unofficial].

**Data Structures:**
1.  **Bricks:** The volumetric dataset is divided into 3D blocks ("bricks") to fit within the graphics card's texture memory and align with its memory architecture [S1, unofficial].
2.  **Texture0 (View Map):** A static 2D texture map storing the view number for each sub-pixel (R, G, B) of the output image. This is pre-calculated based on the lenticular lens geometry and remains constant for all frames [S1, unofficial].
3.  **Texture1 (View Stack):** A single wide rectangular 2D texture map storing all $N$ rendered views vertically next to each other. Each view occupies a horizontal strip of height $H_{view} = H_{LCD} / N$ [S1, unofficial].

**Parameters:**
*   $N$: Number of views. The specific implementation uses $N=9$, but the method applies to any $N$ [S1, unofficial].
*   $f$: Focal distance of the lenticular lenses.
*   $d$: Distance between adjacent view camera positions.
*   Resolution: Each individual view has an effective resolution of $(1/N)$ of the LCD matrix resolution in the horizontal direction [S1, unofficial].

**Steps:**

1.  **Data Partitioning:** Divide the input volumetric dataset into bricks suitable for GPU texture memory [S1, unofficial].
2.  **View Rendering Loop:** For each view index $n$ from $0$ to $N-1$:
    a.  **Frustum Setup:** Define an asymmetric perspective projection frustum for view $n$ [S1, unofficial]. The camera position is offset from the center [S1, unofficial]. The projection coordinates $(x', y')$ for a world point $(x, y, z)$ are calculated as [S1, unofficial]:
        $$
        x' = \frac{(x - n \cdot d) \cdot f}{f - z} + n \cdot d
        $$
        $$
        y' = \frac{y \cdot f}{f - z}
        $$
    b.  **Direct Volume Rendering (DVR):** Render the volume using the defined frustum [S1, unofficial]. Rays are cast from the focal spot through each pixel [S1, unofficial]. The volume rendering integral is approximated by traversing rays in back-to-front order using the over operator [S1, unofficial]:
        $$
        C_{n+1} = \alpha_n \cdot c_n + (1 - \alpha_n) \cdot C_n
        $$
        where $C_n$ is the intermediate color value, $\alpha_n$ is the opacity, and $c_n$ is the color at sample $n$ [S1, unofficial]. This is implemented using standard alpha blending (e.g., DirectX or OpenGL) on textured slices [S1, unofficial].
    c.  **Storage:** Store the resulting image for view $n$ in the $n$-th vertical strip of `Texture1` [S1, unofficial].
3.  **Compositing:** Generate the final interlaced image by parsing every pixel of the output LCD resolution [S1, unofficial]. For each pixel, read the view numbers for the Red, Green, and Blue components from `Texture0` [S1, unofficial]. Sample the corresponding view from `Texture1` for each color channel to construct the final pixel color [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_lenticular_volume(volume_data, lenticular_params, lcd_resolution):
    N = lenticular_params.N
    f = lenticular_params.f
    d = lenticular_params.d
    view_height = lcd_resolution[1] // N
    
    # Step 1: Pre-calculate view map (Texture0)
    # For each pixel (px, py) in the LCD, determine which view index 
    # corresponds to R, G, and B channels based on lens geometry.
    # This is static and computed once.
    view_map = compute_view_map(lcd_resolution, lenticular_params)
    
    # Step 2: Render each view
    # Texture1 is a 2D texture of size (width, height * N)
    texture1 = [[0.0] * (lcd_resolution[0] * lcd_resolution[1] * N) for _ in range(1)]
    
    for n in range(N):
        # Define asymmetric frustum for view n
        frustum = compute_asymmetric_frustum(n, d, f)
        
        # Render volume to a temporary buffer for this view
        view_buffer = [[0.0] * 3 for _ in range(lcd_resolution[0] * view_height)]
        
        # Divide the volumetric dataset into bricks to fit within texture memory [S1, unofficial]
        # Render the volume using the defined frustum. Rays are cast from the focal spot through each pixel.
        # The volume rendering integral is approximated by traversing rays in back-to-front order
        # using the over operator: C_{n+1} = alpha_n * c_n + (1 - alpha_n) * C_n [S1, unofficial]
        # This is implemented using standard alpha blending on textured slices.
        # The sources do not give the exact ray-marching and brick traversal algorithm. [G]
        for brick in volume_data:
            for y in range(view_height):
                for x in range(lcd_resolution[0]):
                    # Simulate alpha blending step for each sample along the ray
                    pass 
        
        # Store view_buffer in the n-th vertical strip of Texture1
        for y in range(view_height):
            for x in range(lcd_resolution[0]):
                idx = x + (n * view_height + y) * lcd_resolution[0]
                texture1[0][idx] = view_buffer[y * lcd_resolution[0] + x]

    # Step 3: Composite final image
    final_image = [[0.0] * 3 for _ in range(lcd_resolution[0] * lcd_resolution[1])]
    
    for py in range(lcd_resolution[1]):
        for px in range(lcd_resolution[0]):
            # Get view indices for R, G, B from the static map
            view_r, view_g, view_b = view_map[px, py]
            
            # Sample from Texture1
            y_r = view_r * view_height + (py % view_height)
            y_g = view_g * view_height + (py % view_height)
            y_b = view_b * view_height + (py % view_height)
            
            idx_r = px + y_r * lcd_resolution[0]
            idx_g = px + y_g * lcd_resolution[0]
            idx_b = px + y_b * lcd_resolution[0]
            
            color_r = texture1[0][idx_r]
            color_g = texture1[0][idx_g]
            color_b = texture1[0][idx_b]
            
            final_image[px + py * lcd_resolution[0]] = (color_r, color_g, color_b)
            
    return final_image

def compute_asymmetric_frustum(n, d, f):
    def project(x, y, z):
        x_proj = ((x - n * d) * f) / (f - z) + n * d
        y_proj = (y * f) / (f - z)
        return (x_proj, y_proj)
    return project

def compute_view_map(lcd_resolution, lenticular_params):
    # Pre-calculates which view index corresponds to R, G, B for each pixel.
    # This depends on the physical alignment of the LCD sub-pixels under the lenticular lenses.
    # Not given in the sources. [G]
    return [[(0, 0, 0)] * lcd_resolution[0] for _ in range(lcd_resolution[1])]
```

**Edge Cases and Pitfalls:**
*   **Memory Limits:** The dataset must be divided into bricks to fit within the graphics card's texture memory [S1, unofficial].
*   **Resolution Trade-off:** Each view has a lower resolution ($1/N$ of LCD width), which may reduce detail per view but allows for interactive frame rates [S1, unofficial].
*   **Static Geometry:** The view map (`Texture0`) is static; it assumes the display and camera setup do not change during runtime [S1, unofficial].

## Subtopics

### [Lenticular Sheet](22655-lenticular-sheet.md)
A refractive optical component consisting of a linear array of thick plano-convex cylindrical lenses (lenticules) used in multiview autostereoscopic displays to direct light from a 2D screen into distinct viewing zones. It generates multi-view stereoscopic images without requiring glasses, acting as a transparent, high-efficiency alternative to parallax barriers. Unlike parallax barriers which block light, lenticular sheets refract it, offering better brightness and higher potential resolution, though it suffers from limited per-view resolution ($1/N$ of the display resolution for $N$ views) and strict alignment requirements. Choose this component when hardware-accelerated glasses-free 3D visualization is required and brightness/resolution trade-offs are acceptable.

## Operating characteristics
*   **Performance:** Up to 15 frames per second for a $256^3$ voxel dataset on a nine-view lenticular display [S1, unofficial].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Hardware-accelerated interactive manipulation of large datasets (>100 MB); no glasses required for the viewer; leverages standard GPU primitives (alpha blending, texture sampling) [S1, unofficial].
*   **Limitations:** Requires specific lenticular hardware; resolution per view is reduced; asymmetric frustum calculations add complexity to the projection step [S1, unofficial].

## Background
The method was proposed by Daniel Ruijters from Philips Medical Systems [S1, unofficial]. It addresses the growing size of volumetric medical imaging data and the need for interactive 3D visualization without glasses [S1, unofficial]. The approach builds on Direct Volume Rendering [2] and multi-view display principles [3, 5, 6, 7] cited in the original paper [S1, unofficial].

## Sources
[S1] GPU-accelerated volume rendering to 3D lenticular displays, https://lirias.kuleuven.be/bitstream/123456789/235668/1/GPU_Volume_Rendering_to_Lenticular_Display7.pdf, fetched 2026-10-27 (unofficial)
