# In-Core Volume Rendering

Also called: Srend, Srend In-Core Volume Rendering, In-core volume rendering for Cartesian grid fluid dynamics simulations

## Overview
In-Core Volume Rendering, specifically implemented as the **Srend** library, is a method for embedding direct volume ray casting directly within computational fluid dynamics (CFD) simulation codes to produce immediate, high-quality imagery without external post-processing [S1]. It solves the problem of visualization overhead and storage requirements in large-scale parallel simulations by performing rendering in-core, utilizing time partitioning where every simulation process renders its own data [S1]. It differs from established volume renderers by lacking a supervisory load-balancing mechanism; instead, it relies on the associativity of pre-multiplied alpha compositing to combine results from distributed processes via a user-defined "rendering tree" [S1].

## How it works
The method operates on the fundamental loop structure of [Direct volume ray casting](19-direct-volume-ray-casting.md): for each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, intersected with the volume domain, and sampled/composited along its path [G]. The traversal order is determined by the pixel's position in the image plane [G].

The core computational engine consists of four sequential stages applied to each ray [S5, S10, S37, unofficial]:

1.  **Ray Generation and Discretization**: The ray segment $[t_{start}, t_{end}]$ intersecting the volume is determined, typically by intersecting the ray with the volume's bounding box [S1][S8, S16, unofficial]. The segment is discretized into $N$ samples at equidistant intervals $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17, unofficial].
2.  **Sampling and Interpolation**: At each sample point $\mathbf{p}_i = r(t_i)$, the scalar value $V(\mathbf{p}_i)$ is retrieved from the volume data. This is typically performed by trilinear interpolation of the eight surrounding voxels [S9, S11, S47, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $V(\mathbf{p}_i)$ is mapped to optical properties: color $C_i$ and opacity $\alpha_i$ [S2, S2789]. The opacity is often corrected for the step size $\Delta t$ to ensure physical consistency, using the Beer-Lambert law approximation:
    $$ \alpha_i = 1 - e^{-\sigma_i \Delta t} $$
    where $\sigma_i$ is the extinction coefficient derived from the scalar value [S24, S30, unofficial].
4.  **Compositing**: The samples are accumulated from front to back (closest to the eye to furthest) using the front-to-back alpha compositing equation [S1][S2, S13, unofficial][S61]:
    $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
    $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
    where $C_{in}$ and $\alpha_{in}$ are the accumulated color and opacity from previous samples, and $C_{sample}$ and $\alpha_{sample}$ are the color and opacity of the current sample [S52, S60, unofficial]. The process initializes with $C_{in} = 0$ and $\alpha_{in} = 0$ [S15, S46, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Volume data: Rectangular "bricks" represented as 3D arrays of variable values [S1].
    *   Data Format: Arrays of bytes (e.g., Fortran `character*1`) scaled to a color table [S1]. The user must scale real simulation data to bytes and define a color table [S1].
    *   Rendering Parameters: View index, camera position/direction, number of samples, and color table definition [S1].
    *   MPI Context: Simulation worker ranks and the user-defined "rendering tree" topology [S1].
*   **Outputs**:
    *   A 2D rendered image (pre-multiplied alpha plane) for the local brick [S1].
    *   Composited final image written to disk by the root finisher [S1].

### Data Structures
*   **Volume Brick**: A 3D array of bytes. The dimensions correspond to the partitioned subset of the global Cartesian grid held by the current MPI rank [S1].
*   **Color Table**: A mapping from byte values (0-255) to RGBA color and opacity values [S1].
*   **Render State**: Contains the accumulated color $C_{in}$ and opacity $\alpha_{in}$ for the current ray [S1].

### Numbered Steps

1.  **Data Partitioning**: The global simulation volume is partitioned into rectangular bricks among simulation worker ranks [S1]. Each rank holds a subset of the data [S1].
2.  **Local Rendering (Per Rank)**:
    1.  For each pixel in the output image:
        2.  Generate ray $r(t)$ based on the view parameters [S1].
        3.  Intersect ray with the local brick's bounding box to determine $[t_{start}, t_{end}]$ [S1].
        4.  If the ray does not intersect the local brick, skip to the next pixel [S1].
        5.  Discretize the segment into $N$ samples with step $\Delta t$ [S1].
        6.  Initialize $C_{in} = 0, \alpha_{in} = 0$ [S1].
        7.  For each sample $i$ from $0$ to $N-1$:
            8.  Calculate sample position $\mathbf{p}_i = r(t_{start} + i \cdot \Delta t)$ [S1].
            9.  Check if $\mathbf{p}_i$ is within the local brick boundaries [S1].
            10. If outside, continue to next sample [S1].
            11. Retrieve byte value $B$ from the local 3D array at $\mathbf{p}_i$ using interpolation (e.g., trilinear) [S1].
            12. Map $B$ to color $C_{sample}$ and opacity $\alpha_{sample}$ using the color table [S1].
            13. Update compositing state:
                $$ C_{in} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
                $$ \alpha_{in} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
            14. If $\alpha_{in} \ge 1.0$, break loop (ray is fully opaque) [S1].
        15. Store $(C_{in}, \alpha_{in})$ as the pixel value for the local brick's 2D render buffer [S1].
3.  **Compositing (Across Ranks)**:
    1.  Each rank sends its 2D rendered brick buffer to a designated "composer" process according to the rendering tree [S1].
    2.  The composer receives 2D renderings from its sources [S1].
    3.  The composer combines the incoming pre-multiplied alpha planes with its own local result (if it is also a renderer) or with previous composited results [S1].
    4.  The composition follows the associativity of alpha compositing: if $R_A$ and $R_B$ are two pre-multiplied alpha images, the composite $R_C$ is calculated pixel-wise [S1]:
        $$ C_C = C_A + (1 - \alpha_A) \cdot C_B $$
        $$ \alpha_C = \alpha_A + (1 - \alpha_A) \cdot \alpha_B $$
    5.  The result is sent up the rendering tree to the next composer or to the root [S1].
4.  **Finalization**:
    1.  The root "finisher" receives the fully composited image from the final composer [S1].
    2.  The finisher writes the image to disk [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Srend In-Core Volume Rendering Implementation

# Constants
MAX_ALPHA = 1.0
EPSILON = 1e-6

# Data Structures
class ColorTable:
    def __init__(self, entries):
        # entries: list of (r, g, b, a) tuples for byte values 0-255
        self.entries = entries

    def get_color(self, byte_val):
        return self.entries[byte_val]

class Brick:
    def __init__(self, data_3d, bounds):
        # data_3d: 3D array of bytes (uint8)
        # bounds: (min_x, min_y, min_z, max_x, max_y, max_z) in grid coordinates
        self.data = data_3d
        self.bounds = bounds

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

    def intersect_brick(self, brick):
        # Returns (t_start, t_end) or None if no intersection
        # Simplified AABB intersection
        min_b, max_b = brick.bounds
        t_min = -float('inf')
        t_max = float('inf')
        
        for i in range(3):
            if abs(self.direction[i]) < EPSILON:
                if self.origin[i] < min_b[i] or self.origin[i] > max_b[i]:
                    return None
            else:
                t1 = (min_b[i] - self.origin[i]) / self.direction[i]
                t2 = (max_b[i] - self.origin[i]) / self.direction[i]
                t_min = max(t_min, min(t1, t2))
                t_max = min(t_max, max(t1, t2))
        
        if t_min > t_max:
            return None
        return (t_min, t_max)

    def sample(self, t):
        return (self.origin[i] + t * self.direction[i] for i in range(3))

def trilinear_interpolate(brick, point):
    # Returns byte value from brick.data at point
    # Point is in continuous coordinates; needs to be mapped to grid indices
    # This is a placeholder for the specific interpolation logic
    # [G] Implementation details for trilinear interpolation on byte arrays
    pass

def render_brick_local(brick, ray, color_table, num_samples):
    # Renders the local brick for a single ray
    intersection = ray.intersect_brick(brick)
    if intersection is None:
        return None, None
    
    t_start, t_end = intersection
    delta_t = (t_end - t_start) / num_samples
    
    acc_color = (0.0, 0.0, 0.0)
    acc_alpha = 0.0
    
    for i in range(num_samples):
        t = t_start + i * delta_t
        point = ray.sample(t)
        
        # Check if point is within brick bounds (continuous)
        # [G] Simplified boundary check; actual implementation needs precise voxel mapping
        if not is_inside_brick(brick, point):
            continue
            
        byte_val = trilinear_interpolate(brick, point)
        r, g, b, a = color_table.get_color(byte_val)
        
        # Pre-multiply color by alpha for compositing
        r_pm = r * a
        g_pm = g * a
        b_pm = b * a
        
        # Front-to-back compositing
        # C_out = C_in + (1 - alpha_in) * C_sample_pm
        # alpha_out = alpha_in + (1 - alpha_in) * alpha_sample
        # Note: Srend uses pre-multiplied alpha planes. 
        # The standard equation for pre-multiplied alpha is:
        # C_out = C_in + (1 - alpha_in) * C_sample_pm
        # alpha_out = alpha_in + (1 - alpha_in) * alpha_sample
        
        acc_color = (
            acc_color[0] + (1 - acc_alpha) * r_pm,
            acc_color[1] + (1 - acc_alpha) * g_pm,
            acc_color[2] + (1 - acc_alpha) * b_pm
        )
        acc_alpha = acc_alpha + (1 - acc_alpha) * a
        
        if acc_alpha >= MAX_ALPHA:
            break
            
    return acc_color, acc_alpha

def is_inside_brick(brick, point):
    min_b, max_b = brick.bounds
    return (min_b[0] <= point[0] <= max_b[0] and
            min_b[1] <= point[1] <= max_b[1] and
            min_b[2] <= point[2] <= max_b[2])

def composite_pre_multiplied_alpha(color_a, alpha_a, color_b, alpha_b):
    # Composite image B over image A
    # C_out = C_A + (1 - alpha_A) * C_B
    # alpha_out = alpha_A + (1 - alpha_A) * alpha_B
    # Note: C_B is assumed to be pre-multiplied by alpha_B already?
    # In Srend, the "2D renderings" are pre-multiplied alpha planes.
    # So C_B in the formula above is actually the pre-multiplied color.
    
    r_out = color_a[0] + (1 - alpha_a) * color_b[0]
    g_out = color_a[1] + (1 - alpha_a) * color_b[1]
    b_out = color_a[2] + (1 - alpha_a) * color_b[2]
    alpha_out = alpha_a + (1 - alpha_a) * alpha_b
    
    return (r_out, g_out, b_out), alpha_out

def srend_render_loop(brick, color_table, num_samples, num_pixels, ray_generator):
    # Local rendering for all pixels
    local_image = []
    for pixel_idx in range(num_pixels):
        ray = ray_generator(pixel_idx)
        color, alpha = render_brick_local(brick, ray, color_table, num_samples)
        if color is None:
            # Ray did not hit brick, transparent
            local_image.append(((0,0,0), 0.0))
        else:
            local_image.append((color, alpha))
    return local_image

def srend_compose_tree(local_images, rendering_tree):
    # Compose images according to the rendering tree
    # Each node in the tree has a list of children (sources)
    # If a node is also a renderer, it combines its local image with children
    
    # This is a high-level description of the MPI communication
    # [G] Specific MPI send/receive logic is abstracted here
    pass

# Main Execution Flow
def main():
    # 1. Setup
    brick = load_brick_data() # 3D byte array
    color_table = load_color_table() # Map byte to RGBA
    num_samples = 64 # Typical value
    
    # 2. Local Rendering
    local_image = srend_render_loop(brick, color_table, num_samples, num_pixels, generate_rays)
    
    # 3. Send to Composer
    send_to_composer(local_image)
    
    # 4. Receive Composite (if composer)
    # 5. Finalize (if root)
    pass
```

### Parameters
*   **Number of Samples ($N$)**: Not specified in sources. Typical values for ray casting range from 32 to 256 depending on required quality and performance [S1].
*   **Data Scaling**: The user must scale real simulation data to bytes (0-255) [S1].
*   **Color Table**: User-defined mapping from byte values to RGBA [S1].
*   **Rendering Tree**: User-defined topology for MPI compositing [S1].

### Edge Cases and Pitfalls
*   **Grid Interpolation**: If the simulation uses stretched grids or terrain-following vertical coordinates, data arrays must be interpolated to spatial arrays for Srend to maintain spatial accuracy [S1].
*   **Redundant Information**: Every renderer sends identical image finishing parameters (e.g., output file name) up the rendering tree, even though only the finisher uses them. This is a design choice to simplify usage [S1].
*   **No Load Balancing**: Srend has no load balancing or supervisory mechanism. Performance depends on the uniformity of rendering work across ranks [S1].
*   **Data Format**: Data must be in arrays of bytes. Floating-point data must be pre-scaled by the user [S1].

## Operating characteristics
*   **Time Complexity**: Rendering completion time depends on the rendering work itself. There is no synchronization or management overhead beyond that imposed by MPI messaging [S1].
*   **Memory Complexity**: Data is stored in-core. The method eliminates the need for external visualization tools or post-processing steps, reducing data output and storage requirements [S1].
*   **Measured Performance**: Results show modest rendering overhead, fine quality imagery, and high potential for scaling [S1].
*   **Strengths**:
    *   Immediate, quality imagery during simulation [S1].
    *   No reliance on special visualization hardware, libraries, or external programs [S1].
    *   Embarrassingly parallel ray casting [S1].
    *   Scalable via MPI and associativity of compositing [S1].
*   **Limitations**:
    *   Requires user to define color tables and scale data [S1].
    *   No load balancing [S1].
    *   Requires Fortran compatibility for easiest integration (though not strictly required for the algorithm) [S1].
*   **When to Use**: Embedded visualization in CFD simulations using Cartesian grids, particularly when immediate feedback is needed and storage is a concern [S1].
*   **When Not to Use**: When load balancing is critical or when the simulation code is not Fortran-based and integration is difficult [S1].

## Background
*   **Origin**: Srend was developed at the University of Minnesota / LCSE [S1].
*   **History**: It was modified from its original form used in a terrain flow code for the St. Louis River region and redesigned for spherical data rendering (stellar data) [S1]. It was embedded in three codes: PPMstar, CM1, and WRF [S1].
*   **Usage**: Used for visualization of computational fluid dynamics simulations using Cartesian grids [S1].

## Sources
[S1] In-core volume rendering for Cartesian grid fluid dynamics simulations, https://doi.org/10.1145/2792745.2792780, fetched 2026-11-05
