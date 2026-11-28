# In-Core Volume Rendering

Also called: In-core volume rendering for Cartesian grid fluid dynamics simulations, Srend, Srend In-Core Volume Rendering

## Overview
In-core volume rendering, specifically implemented as **Srend**, is a method for visualizing computational fluid dynamics (CFD) simulations using Cartesian grids by compiling the rendering code directly into the simulation application [S1]. It solves the problem of high data I/O overhead and latency in post-processing by generating images in real-time during the simulation [S1]. Unlike general-purpose in-situ visualization frameworks that often employ load balancing or supervisory mechanisms, Srend relies on a user-defined "rendering tree" and time-partitioned rendering where every simulation process renders its own data brick, followed by space-partitioned compositing by dedicated processes [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Simulation Data:** 3D arrays of variable values representing rectangular "bricks" of the volume [S1].
*   **Data Format:** The source data must be scaled to arrays of bytes (Fortran `character*1`) and mapped to a user-defined color table [S1].
*   **Rendering Parameters:** View index, number of sources, MPI target rank, and image finishing parameters (e.g., output file name) [S1].
*   **Geometry:** Spatial coordinates for interpolation if the simulation uses stretched grids or terrain-following coordinates [S1].

**Outputs:**
*   **Final Image:** A 2D rendered image written to disk by the finisher at the root of the rendering tree [S1].

### Data Structures
*   **Brick:** A rectangular sub-volume represented by a 3D array of byte values [S1].
*   **Rendering Tree:** A user-defined hierarchy of MPI ranks where leaves are simulation worker ranks, internal nodes are "composers", and the root is the "finisher" [S1].
*   **Message Payload:** Contains the 2D rendered pre-multiplied alpha plane, view index, number of sources, and redundant image finishing parameters (file name) [S1].

### Numbered Steps
1.  **Data Partitioning:** The simulation domain is partitioned into rectangular bricks, and each simulation worker rank holds one or more bricks [S1].
2.  **Local Rendering (Time Partitioning):** Each worker rank renders its assigned brick using all available threads as if the brick were the entire volume [S1].
    *   The rendering uses ray casting, collecting emissions along rays from the eye through the volume [S1].
    *   The result is a 2D pre-multiplied alpha plane [S1].
3.  **Compositing (Space Partitioning):** Composers receive 2D renderings from their child ranks [S1].
    *   Composers exploit the associativity of compositing pre-multiplied alpha planes to merge the incoming planes with their own local render (if they are also workers) or with previously composited results [S1].
    *   The composited result is sent to the parent rank in the rendering tree [S1].
4.  **Finalization:** The finisher at the root of the rendering tree receives the final composited image and writes it to disk [S1].

### Formulas
The rendering process uses standard ray casting for volume rendering [S1]. The compositing of pre-multiplied alpha planes relies on the associativity of the compositing operator [S1]. For two adjacent alpha planes $A$ and $B$ (where $A$ is closer to the eye), the composite $C$ is calculated as:
$$ C = A + (1 - \alpha_A) B $$
where $\alpha_A$ is the alpha channel of $A$, and $B$ is the pre-multiplied color/alpha of the further plane [S1].

### Parameters
*   **Data Scaling:** Users must manually scale real-valued simulation data to bytes and define a color table [S1].
*   **View Index:** Identifies the camera perspective [S1].
*   **Number of Sources:** Used by composers to track how many children have sent data [S1].
*   **MPI Target Rank:** The destination rank for sending composited results [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Srend In-Core Volume Rendering Implementation

# Standard math/vector operations
def vec4(r, g, b, a): return (r, g, b, a)
def zeros(shape):
    # Returns a 2D list of vec4 initialized to (0,0,0,0)
    return [[vec4(0, 0, 0, 0) for _ in range(shape[1])] for _ in range(shape[0])]

def compute_ray(pixel_x, pixel_y, view_params):
    # Returns (origin: vec3, direction: vec3) based on camera parameters [G]
    return view_params['origin'], view_params['directions'][pixel_y][pixel_x]

def ray_brick_intersection(ray_origin, ray_dir, brick_bounds):
    # Returns (t_min, t_max) for ray intersecting brick axis-aligned bounds [G]
    t_min, t_max = 0.0, float('inf')
    for i in range(3):
        if ray_dir[i] == 0:
            if ray_origin[i] < brick_bounds[i*2] or ray_origin[i] > brick_bounds[i*2+1]:
                return None, None
        else:
            t1 = (brick_bounds[i*2] - ray_origin[i]) / ray_dir[i]
            t2 = (brick_bounds[i*2+1] - ray_origin[i]) / ray_dir[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return t_min, t_max

def interpolate_scalar(ray_origin, ray_dir, t, brick_data, grid_geometry):
    # Interpolates byte value from 3D brick at ray position [G]
    # Assumes grid_geometry maps brick indices to world coordinates
    # Returns a byte value (0-255)
    return 128 # Placeholder for interpolation logic [G]

def composite_alpha(base, source):
    # Composites source onto base using pre-multiplied alpha [S1]
    # base is closer to the eye. Both are 2D lists of vec4.
    result = zeros((len(base), len(base[0])))
    for y in range(len(base)):
        for x in range(len(base[y])):
            a_base = base[y][x][3] / 255.0
            r_base, g_base, b_base, a_base = base[y][x]
            r_src, g_src, b_src, a_src = source[y][x]
            a_src_norm = a_src / 255.0
            # C = A + (1 - alpha_A) * B
            r_new = r_base + (1.0 - a_base) * r_src
            g_new = g_base + (1.0 - a_base) * g_src
            b_new = b_base + (1.0 - a_base) * b_src
            a_new = a_base + (1.0 - a_base) * a_src_norm
            result[y][x] = vec4(r_new, g_new, b_new, a_new * 255.0)
    return result

def render_brick(brick_data_3d, view_params, color_table, brick_bounds, grid_geometry):
    # Renders a single brick using ray casting [S1]
    image_height = view_params['height']
    image_width = view_params['width']
    image = zeros((image_height, image_width))
    ray_step_size = view_params['step_size'] # [G]

    for y in range(image_height):
        for x in range(image_width):
            origin, direction = compute_ray(x, y, view_params)
            t_start, t_end = ray_brick_intersection(origin, direction, brick_bounds)
            if t_start is None:
                continue

            color_accum = vec4(0, 0, 0, 0)
            t = t_start
            while t < t_end:
                val = interpolate_scalar(origin, direction, t, brick_data_3d, grid_geometry)
                rgba = color_table[val] # Returns vec4
                # Front-to-back compositing for pre-multiplied alpha [S1]
                a_accum = color_accum[3] / 255.0
                r_new = color_accum[0] + (1.0 - a_accum) * rgba[0]
                g_new = color_accum[1] + (1.0 - a_accum) * rgba[1]
                b_new = color_accum[2] + (1.0 - a_accum) * rgba[2]
                a_new = a_accum + (1.0 - a_accum) * (rgba[3] / 255.0)
                color_accum = vec4(r_new, g_new, b_new, a_new * 255.0)
                t += ray_step_size
            image[y][x] = color_accum
    return image

def srend_worker(brick_data, view_index, num_sources, target_rank, color_table, view_params, brick_bounds, grid_geometry):
    # Executed by simulation worker ranks [S1]
    local_image = render_brick(brick_data, view_params, color_table, brick_bounds, grid_geometry)
    payload = {
        'image': local_image,
        'view_index': view_index,
        'num_sources': num_sources,
        'file_name': 'output.png' # Redundant but simplifies usage [S1]
    }
    # MPI non-blocking send [S1]
    mpi_send(target_rank, payload)

def srend_composer(child_ranks, view_index, num_sources, parent_rank, color_table, view_params, brick_bounds, grid_geometry):
    # Executed by intermediate ranks in the rendering tree [S1]
    composite_image = zeros((view_params['height'], view_params['width']))
    for child_rank in child_ranks:
        msg = mpi_recv(child_rank)
        composite_image = composite_alpha(composite_image, msg['image'])
    # If this rank also holds data (hybrid worker/composer), render and composite [S1]
    if has_local_brick():
        local_image = render_brick(get_local_brick(), view_params, color_table, brick_bounds, grid_geometry)
        composite_image = composite_alpha(composite_image, local_image)
    if parent_rank is not None:
        payload = {
            'image': composite_image,
            'view_index': view_index,
            'num_sources': num_sources,
            'file_name': 'output.png'
        }
        mpi_send(parent_rank, payload)

def srend_finisher(child_ranks, view_index, num_sources, output_filename, view_params):
    # Executed by the root rank [S1]
    composite_image = zeros((view_params['height'], view_params['width']))
    for child_rank in child_ranks:
        msg = mpi_recv(child_rank)
        composite_image = composite_alpha(composite_image, msg['image'])
    # Write final image to disk [S1]
    save_image(output_filename, composite_image)

# MPI stubs for self-containment
def mpi_send(rank, data): pass
def mpi_recv(rank): return {'image': zeros((1,1)), 'view_index': 0, 'num_sources': 0, 'file_name': ''}
def has_local_brick(): return False
def get_local_brick(): return []
def save_image(filename, data): pass
```

### Edge Cases and Pitfalls
*   **Grid Interpolation:** If the simulation uses stretched grids or terrain-following vertical coordinates, data arrays must be interpolated to spatial arrays for Srend to maintain spatial accuracy [S1].
*   **Redundant Information:** The output file name and other image finishing parameters are sent by every renderer, even though only the finisher uses them [S1]. This is a design choice to simplify usage, accepting a light data payload overhead [S1].
*   **Data Scaling:** Users must manually scale real data to bytes and define a color table [S1]. This requires knowledge of the data distribution and appearance requirements [S1].

## Operating characteristics
*   **Time Complexity:** Rendering completion time (simulation halt time) depends primarily on the rendering work itself [S1]. There is no synchronization or management overhead beyond that imposed by MPI messaging [S1].
*   **Memory Complexity:** Data is kept in-core [S1]. The memory footprint includes the simulation data and the intermediate 2D rendered planes [S1].
*   **Performance:** Results show modest rendering overhead and high potential for scaling [S1]. The 3D data is already in CPU memory, requiring only movement to registers, making it efficient on CPUs compared to GPUs for this specific workflow where cluster communications are well-established for the lighter 2D results [S1].
*   **Strengths:** Immediate, quality imagery; reduces data output and storage requirements; no reliance on special visualization hardware or external programs; convenient integration into Fortran simulation codes [S1].
*   **Limitations:** No load balancing or supervisory mechanism; rendering and compositing are defined by the user's incorporation of Srend calls; requires manual data scaling and color table definition [S1].
*   **When to Use:** When embedding visualization directly into Fortran-based CFD simulations (e.g., PPMstar, CM1, WRF) to reduce I/O overhead and enable real-time exploration [S1].
*   **When Not to Use:** When dynamic load balancing is required or when the simulation code is not in Fortran or cannot be easily modified to include Srend calls [S1].

## Background
Srend was developed at the University of Minnesota / LCSE [S1]. It was modified from its original form, which was used within a terrain flow code developed to study the advection of smoke and pollen in the St. Louis River region [S1]. The "S" in Srend originally stood for spherical data rendering, supporting longitude/latitude views for full spherical rendering of stellar data from the center, though perspective views and other features were retained for general use [S1]. It was embedded in three major codes: Piecewise Parabolic Method (PPMstar), Cloud Model 1 (CM1), and Weather Research Forecast model (WRF) [S1]. The code was implemented in Fortran to allow convenient inclusion within Fortran simulation codes using ordinary subroutine calls and familiar Fortran arrays [S1].

## Sources
[S1] In-core volume rendering for Cartesian grid fluid dynamics simulations, https://doi.org/10.1145/2792745.2792780, fetched 2026-11-05
