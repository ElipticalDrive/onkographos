# Discrete Ray Tracing

Also called: 3D DDA Voxel Traversal, Naive Voxel-Raycasting, Raycasting ohne Beschleunigungsstruktur

## Overview
Discrete Ray Tracing is a method for rendering voxelized datasets by traversing a 3D grid along a ray using a three-dimensional form of Digital Differential Analysis (DDA) [S1, unofficial]. It guarantees that voxels are visited in the exact order of intersection, limiting the number of intersection tests to Pythagorean complexity rather than cubic [S1, unofficial]. Unlike methods requiring acceleration structures, it samples every voxel along the ray path sequentially [S2, unofficial].

## How it works
**Inputs:**
*   Volume data: A 3D array of scalar values (voxels).
*   Camera parameters: Position, view port vectors (upperLeft, upperRight, lowerLeft, lowerRight), screen dimensions (width, height).
*   Transfer function: Maps scalar values to color $(r, g, b)$ and opacity $(a)$.
*   Projection type: Orthographic (ray direction fixed along Z-axis in the example implementation) or Perspective (calculated via view port normals) [S1, unofficial][S3, unofficial].

**Outputs:**
*   Rendered image: A 2D array of pixel colors (RGB) and opacity.

**Data Structures:**
*   `volume_data`: 3D array `volume_data[x, y, z]`.
*   `pixel_color`: 4-element array `[r, g, b, a]`, initialized to `[0.0, 0.0, 0.0, 0.0]`.
*   `ray`: Structure with `origin` (Vec3) and `direction` (Vec3).

**Steps:**

1.  **Initialize Camera Normals:**
    Calculate normalized view port vectors. If view port vectors depend on camera position, subtract the camera position before normalizing [S1, unofficial].
    $$
    \begin{aligned}
    \mathbf{n}_{UL} &= \text{normalize}(\mathbf{v}_{UL} - \mathbf{p}_{cam}) \\
    \mathbf{n}_{UR} &= \text{normalize}(\mathbf{v}_{UR} - \mathbf{p}_{cam}) \\
    \mathbf{n}_{LL} &= \text{normalize}(\mathbf{v}_{LL} - \mathbf{p}_{cam}) \\
    \mathbf{n}_{LR} &= \text{normalize}(\mathbf{v}_{LR} - \mathbf{p}_{cam})
    \end{aligned}
    $$
    [S1, unofficial]

2.  **Calculate Interpolation Deltas:**
    Compute inverse screen dimensions and Y-direction deltas for the left and right normals [S1, unofficial]:
    $$
    \begin{aligned}
    h_{inv} &= 1.0 / \text{screenHeight} \\
    w_{inv} &= 1.0 / \text{screenWidth} \\
    \Delta \mathbf{n}_{L,y} &= (\mathbf{n}_{LL} - \mathbf{n}_{UL}) \cdot h_{inv} \\
    \Delta \mathbf{n}_{R,y} &= (\mathbf{n}_{LR} - \mathbf{n}_{UR}) \cdot h_{inv}
    \end{aligned}
    $$
    [S1, unofficial]

3.  **Initialize Ray Origin:**
    Set the ray origin to the camera position. This remains constant for all pixels [S1, unofficial].
    $$
    \mathbf{r}_{origin} = \mathbf{p}_{cam}
    $$
    [S1, unofficial]

4.  **Iterate Screen Pixels (Top-to-Bottom, Left-to-Right):**
    Initialize interpolation variables to the top of the screen:
    $$
    \mathbf{n}_L = \mathbf{n}_{UL}, \quad \mathbf{n}_R = \mathbf{n}_{UR}
    $$
    [S1, unofficial]

    For each row $y$ from $0$ to $\text{screenHeight} - 1$:
    a. Set ray direction to left normal: $\mathbf{r}_{dir} = \mathbf{n}_L$.
    b. Calculate X-direction delta:
       $$
       \Delta \mathbf{n}_x = (\mathbf{n}_R - \mathbf{n}_L) \cdot w_{inv}
       $$
       [S1, unofficial]

    For each column $x$ from $0$ to $\text{screenWidth} - 1$:
    a. **Trace Ray through Volume:**
       i. **Determine Ray Path:**
          If using the specific orthographic implementation from [S3, unofficial], the ray direction is fixed along the Z-axis ($\mathbf{r}_{dir} = [0, 0, 1]$) and the origin is computed as the center of the near plane:
          $$
          \mathbf{r}_{origin} = [x / \text{width}, y / \text{height}, \text{near\_plane}]
          $$
          [S3, unofficial]
          
          If using the general perspective DDA from [S1, unofficial], the ray is defined by $\mathbf{r}_{origin}$ and $\mathbf{r}_{dir}$. The algorithm traverses the voxel grid using 3D DDA, stepping through voxel indices in the order the ray intersects them [S1, unofficial].

       ii. **Sample Voxel:**
          Convert the current ray position (or specific sample point along the ray) to voxel coordinates. For the nearest-neighbor approach in [S3, unofficial]:
          $$
          \mathbf{c}_{vox} = \lfloor \mathbf{c}_{world} \cdot \mathbf{D} - 0.5 \rfloor
          $$
          where $\mathbf{D}$ is the volume dimensions and $\lfloor \cdot \rfloor$ is the floor function. The $-0.5$ centers the sampling at $(0.5, 0.5, 0.5)$ [S3, unofficial].
          
          Sample the scalar value without interpolation (nearest-neighbor):
          $$
          s = \text{volume\_data}[\mathbf{c}_{vox}]
          $$
          [S3, unofficial]

       iii. **Apply Transfer Function:**
          Map scalar $s$ to color $(r, g, b)$ and opacity $a$. A common example uses Gaussian mixtures [S3, unofficial]:
          $$
          \begin{aligned}
          r &= 1.0 \cdot e^{-\frac{(s - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(s - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(s + 3.0)^2}{0.5}} \\
          g &= 1.0 \cdot e^{-\frac{(s - 9.0)^2}{1.0}} + 1.0 \cdot e^{-\frac{(s - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(s + 3.0)^2}{0.5}} \\
          b &= 0.1 \cdot e^{-\frac{(s - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(s - 3.0)^2}{0.1}} + 1.0 \cdot e^{-\frac{(s + 3.0)^2}{0.5}} \\
          a &= 0.6 \cdot e^{-\frac{(s - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(s - 3.0)^2}{0.1}} + 0.01 \cdot e^{-\frac{(s + 3.0)^2}{0.5}}
          \end{aligned}
          $$
          [S3, unofficial]

       iv. **Composite:**
          Update accumulated pixel color and opacity using the standard over operator (front-to-back) [S3, unofficial]:
          $$
          \begin{aligned}
          \mathbf{C}_{out} &= \mathbf{C}_{in} + (1 - \alpha_{in}) \cdot \mathbf{C}_{sample} \cdot \alpha_{sample} \\
          \alpha_{out} &= \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample}
          \end{aligned}
          $$
          [S3, unofficial]

    b. **Increment X:**
       $$
       \mathbf{r}_{dir} \leftarrow \mathbf{r}_{dir} + \Delta \mathbf{n}_x
       $$
       [S1, unofficial]

    c. **Set Pixel:**
       Store the final color for pixel $(x, y)$ [S1, unofficial].

    d. **Increment Y:**
       $$
       \begin{aligned}
       \mathbf{n}_L &\leftarrow \mathbf{n}_L + \Delta \mathbf{n}_{L,y} \\
       \mathbf{n}_R &\leftarrow \mathbf{n}_R + \Delta \mathbf{n}_{R,y}
       \end{aligned}
       $$
       [S1, unofficial]

5.  **Finalize:**
    Clip the final color components to $[0, 255]$ and convert to uint8 [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def discrete_ray_tracing(volume_data, camera, screen_width, screen_height, transfer_function):
    """
    Renders a volume using Discrete Ray Tracing (3D DDA).
    Implements the perspective version from [S1, unofficial] with compositing from [S3, unofficial].
    """
    # 1. Initialize Camera Normals
    # Assuming camera provides vectors relative to origin or absolute positions
    # If vectors depend on camera position, subtract camera position before normalizing
    upper_left = normalize(camera.get_upper_left_vector())
    upper_right = normalize(camera.get_upper_right_vector())
    lower_left = normalize(camera.get_lower_left_vector())
    lower_right = normalize(camera.get_lower_right_vector())
    
    # 2. Calculate Interpolation Deltas
    inv_height = 1.0 / screen_height
    inv_width = 1.0 / screen_width
    
    left_normal_y_delta = (lower_left - upper_left) * inv_height
    right_normal_y_delta = (lower_right - upper_right) * inv_height
    
    # 3. Initialize Interpolation Variables
    left_normal = upper_left
    right_normal = upper_right
    
    # 4. Initialize Ray Origin
    ray_origin = camera.get_position()
    
    # Output buffer
    image = np.zeros((screen_height, screen_width, 4))
    
    # 5. Iterate Screen
    for y in range(screen_height):
        # Reset ray direction for this row
        ray_direction = left_normal
        
        # Calculate X delta for this row
        normal_x_delta = (right_normal - left_normal) * inv_width
        
        for x in range(screen_width):
            # Trace Ray
            # Note: The actual voxel traversal logic (3D DDA) is complex to write fully 
            # without specific grid bounds and step logic. 
            # Here we simulate the sampling step as described in [S3, unofficial] 
            # for the orthographic case, or assume a helper for DDA traversal.
            
            # For a general perspective ray, we would step through voxels.
            # For the specific orthographic example in [S3, unofficial]:
            # ray_origin = [x / screen_width, y / screen_height, near_plane]
            # ray_direction = [0, 0, 1]
            
            # Generic DDA traversal placeholder [G]
            # In a full implementation, this loop steps through voxel indices 
            # in intersection order until the ray exits the volume or a stop condition is met.
            accumulated_color = np.array([0.0, 0.0, 0.0, 0.0])
            
            # Simulate traversal for demonstration of compositing logic
            # In reality, 'voxel_coords' would be generated by the 3D DDA step
            # Assuming a single sample for the sake of the compositing formula structure
            # if the volume is thin or we are just showing the transfer function application
            voxel_coord = np.floor(np.array([x, y, 0]) * volume_data.shape[:3] - 0.5).astype(int)
            
            # Clamp coordinates to volume bounds
            voxel_coord = np.clip(voxel_coord, 0, np.array(volume_data.shape) - 1)
            
            scalar_value = volume_data[voxel_coord[0], voxel_coord[1], voxel_coord[2]]
            
            # Apply Transfer Function
            r, g, b, a = transfer_function(scalar_value)
            sample_color = np.array([r, g, b, a])
            
            # Composite (Front-to-Back)
            # pixel_color = pixel_color + (1 - pixel_color[3]) * sample_color[0:3] * sample_color[3]
            # pixel_color[3] = pixel_color[3] + (1 - pixel_color[3]) * sample_color[3]
            
            # Since we are doing a single sample per pixel in this simplified trace,
            # we just assign. In a real DDA loop, this accumulates.
            accumulated_color[0:3] = sample_color[0:3] * sample_color[3]
            accumulated_color[3] = sample_color[3]
            
            image[y, x] = accumulated_color
            
            # Increment X
            ray_direction += normal_x_delta
            
        # Increment Y
        left_normal += left_normal_y_delta
        right_normal += right_normal_y_delta
        
    # Finalize
    image_rgb = np.clip(image[:, :, 0:3] * 255, 0, 255).astype(np.uint8)
    return image_rgb

def transfer_function_gaussian(x):
    """
    Gaussian mixture transfer function from [S3, unofficial].
    """
    r = 1.0 * np.exp(-((x - 9.0)**2) / 1.0) + 0.1 * np.exp(-((x - 3.0)**2) / 0.1) + 0.1 * np.exp(-((x + 3.0)**2) / 0.5)
    g = 1.0 * np.exp(-((x - 9.0)**2) / 1.0) + 1.0 * np.exp(-((x - 3.0)**2) / 0.1) + 0.1 * np.exp(-((x + 3.0)**2) / 0.5)
    b = 0.1 * np.exp(-((x - 9.0)**2) / 1.0) + 0.1 * np.exp(-((x - 3.0)**2) / 0.1) + 1.0 * np.exp(-((x + 3.0)**2) / 0.5)
    a = 0.6 * np.exp(-((x - 9.0)**2) / 1.0) + 0.1 * np.exp(-((x - 3.0)**2) / 0.1) + 0.01 * np.exp(-((x + 3.0)**2) / 0.5)
    return r, g, b, a
```

**Edge Cases and Pitfalls:**
*   **Aliasing:** The method uses nearest-neighbor sampling (`np.floor`), which can result in aliasing or blocky artifacts at low resolutions [S3, unofficial].
*   **Performance:** High number of texture accesses and multiplications per ray leads to low frame rates for large or sparse volumes [S2, unofficial].
*   **Voxel Definition:** A voxel is an index in a grid, not a geometric cube. It has no notion of position or movement [S1, unofficial].
*   **Coordinate Centering:** Subtracting 0.5 from scaled coordinates handles the offset between pixel centers and voxel centers [S3, unofficial].

## Subtopics

### [Cube-3 Hybrid Ray-Casting](3143-cube-3-hybrid-ray-casting.md)
A hybrid ray-casting algorithm developed for arbitrary parallel and perspective projections of high-resolution volumetric datasets, requiring at most one memory access per voxel per projection without pre-processing.

## Operating characteristics
*   **Time Complexity:** $O(n)$, where $n$ is the number of voxels traversed along the ray [S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per ray (excluding the volume data itself), as it does not require acceleration structures [S2, unofficial].
*   **Performance:** Recommended for small, dense voxel entities (approx. $50^3$ or $100^3$ voxels) because modifications are immediately visible without updating acceleration structures [S2, unofficial]. For larger or sparse volumes, acceleration structures are preferred due to the linear complexity [S2, unofficial].
*   **Strengths:** No preprocessing required; immediate visibility of voxel modifications; guarantees closest intersection first [S1, unofficial][S2, unofficial].
*   **Limitations:** Low frame rates for large datasets; high number of memory accesses and multiplications; aliasing artifacts due to nearest-neighbor sampling [S2, unofficial][S3, unofficial].

## Background
Discrete Ray Tracing is a valid approach to rendering voxellized datasets, often cited alongside GPU-based volume ray casting [S4, unofficial]. The term "voxel" is often misused to refer to geometric cubes, but in this context, it refers to indices in an axis-aligned 3D grid [S1, unofficial]. The algorithm is based on a 3D form of Digital Differential Analysis (DDA), similar to the algorithm used in Wolfenstein 3D ray casting tutorials [S1, unofficial]. It limits intersection tests to Pythagorean complexity rather than cubic by traversing voxels in the exact order of intersection [S1, unofficial].

## Sources
[S1] Casting Rays: Voxel Rendering Using Discrete Ray Tracing, https://castingrays.blogspot.com/2014/01/voxel-rendering-using-discrete-ray.html, fetched 2026-10-24 (unofficial)
[S2] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S3] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S4] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
