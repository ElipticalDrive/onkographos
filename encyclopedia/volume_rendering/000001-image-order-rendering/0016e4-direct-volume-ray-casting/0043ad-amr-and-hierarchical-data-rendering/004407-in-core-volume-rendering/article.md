# In-Core Volume Rendering

Also called: Srend, Srend In-Core Volume Rendering, In-core volume rendering for Cartesian grid fluid dynamics simulations

## Overview
In-core volume rendering, specifically as implemented in the **Srend** system, is a method for visualizing Computational Fluid Dynamics (CFD) simulations using Cartesian grids by embedding the rendering code directly within the simulation application [S1]. It solves the problem of high data I/O overhead and latency in large-scale parallel simulations by performing rendering in-core during the simulation run, eliminating the need for external visualization tools or post-processing steps [S1]. Unlike general-purpose in-situ renderers that often employ complex load-balancing or supervisory mechanisms, Srend relies on a simple time-partitioning strategy where every simulation process renders its local data, followed by space-partitioned compositing via a user-defined "rendering tree" [S1]. This approach exploits the embarrassingly parallel nature of ray casting and the associativity of pre-multiplied alpha compositing to achieve scalable, low-overhead visualization [S1].

## How it works
The method integrates ray casting, local rendering, and distributed compositing into the simulation loop. It is designed as a Fortran library to be compiled directly into simulation codes (e.g., PPMstar, CM1, WRF) [S1].

### Inputs and Outputs
*   **Inputs**:
    *   Simulation state data: 3D arrays of variable values (scalar fields) representing the volume [S1].
    *   Grid geometry: Cartesian grid coordinates. For non-uniform grids (stretched or terrain-following), spatial coordinates must be interpolated to ensure spatial accuracy [S1].
    *   Rendering parameters: View index, camera position, projection type (perspective or orthographic), and color/opacity transfer function definitions [S1].
    *   Data format: The scalar values must be pre-scaled to 8-bit unsigned integers (bytes) and mapped to a user-defined color table [S1].
*   **Outputs**:
    *   A final 2D image file (e.g., PNG, TIFF) written to disk by the root finisher [S1].
    *   Intermediate 2D pre-multiplied alpha planes sent between processes [S1].

### Data Structures
The data is partitioned into rectangular "bricks" [S1].
*   **Brick**: A 3D array of bytes (`character*1` in Fortran) representing a sub-volume of the simulation domain [S1].
*   **Color Table**: A user-defined mapping from byte values (0-255) to RGBA colors [S1].
*   **Rendering Tree**: A logical hierarchy of MPI ranks defined by the user to specify how 2D renderings are composited. It is not a spatial data structure but a communication topology [S1].

### Numbered Steps

1.  **Data Partitioning**: The global simulation domain is divided among MPI worker ranks. Each rank holds a local "brick" of data (a 3D array of bytes) [S1].
2.  **Local Rendering (Ray Casting)**:
    *   Each MPI rank performs volume ray casting on its local brick independently [S1].
    *   Rays are generated from the eye position through the volume of interest [S1].
    *   For each ray, the algorithm samples the local brick. If the ray passes through regions not owned by the current rank, those regions are treated as empty or transparent (the rank renders "as if the brick was alone in space") [S1].
    *   The ray casting loop accumulates color and opacity using front-to-back compositing (Beer-Lambert law) to produce a 2D pre-multiplied alpha plane for the brick [S1].
    *   *Note*: The sources state that data must be scaled to bytes and a color table defined by the user. The interpolation kernel depends on the grid type; for Cartesian grids, standard trilinear or nearest-neighbor interpolation is implied, but for stretched/terrain-following grids, spatial interpolation is required [S1].
3.  **Compositing (Tree Reduction)**:
    *   Each rank sends its 2D rendered alpha plane to a designated "composer" rank according to the user-defined rendering tree [S1].
    *   Composers receive 2D renderings from their children in the tree.
    *   Composers perform pixel-wise pre-multiplied alpha compositing of the received planes [S1].
    *   The compositing operation is associative, allowing arbitrary tree structures [S1].
    *   The result is sent up the tree toward the root [S1].
4.  **Finalization**:
    *   The root rank (finisher) receives the fully composited 2D image from the final composer [S1].
    *   The finisher writes the image to disk [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Srend In-Core Volume Rendering Implementation
# Note: This is a conceptual Python representation of the Fortran-based Srend logic.
# The actual implementation is embedded in Fortran simulation codes.

import mpi4py.MPI as MPI
import numpy as np

# Global configuration
comm = MPI.COMM_WORLD
rank = comm.Get_rank()
size = comm.Get_size()

# User-defined parameters (passed in each rendering call)
view_index = 0
num_sources = 0  # Number of child ranks in the rendering tree for this node
target_rank = -1 # MPI rank to send composited result to
output_filename = "rendering.png"

def scale_data_to_bytes(simulation_data, min_val, max_val):
    """
    User-defined step: Scale real simulation data to 0-255 bytes.
    Requires user knowledge of data distribution.
    """
    # [G] Standard min-max normalization clamped to 0-255
    normalized = (simulation_data - min_val) / (max_val - min_val)
    normalized = np.clip(normalized, 0, 1)
    return (normalized * 255).astype(np.uint8)

def define_color_table():
    """
    User-defined step: Define mapping from byte value to RGBA.
    Returns a 256x4 array.
    """
    # [G] Placeholder for user-defined transfer function
    return np.zeros((256, 4), dtype=np.float32)

def ray_cast_brick(brick_data, color_table, view_params):
    """
    Local ray casting for a single brick.
    brick_data: 3D numpy array of uint8 (local sub-volume)
    color_table: 256x4 array of RGBA
    view_params: Contains eye position, direction, etc.
    Returns: 2D pre-multiplied alpha plane (height x width x 4)
    """
    height, width = view_params['image_dims']
    # Initialize output buffer with 0 opacity, 0 color
    alpha_plane = np.zeros((height, width, 4), dtype=np.float32)
    
    # Ray generation and casting loop
    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = view_params.get_ray(x, y)
            
            # Front-to-back compositing
            accumulated_color = np.array([0.0, 0.0, 0.0, 0.0])
            accumulated_alpha = 0.0
            
            # Sample along ray
            # [G] Simplified ray marching; actual Srend uses optimized ray traversal
            t = 0.0
            while t < view_params['max_dist']:
                # Interpolate value from brick_data at current ray position
                # [G] Assumes brick_data covers the ray segment or returns 0 if out of bounds
                voxel_value = interpolate(brick_data, ray_origin + t * ray_direction)
                
                # Map voxel value to RGBA using color table
                rgba = color_table[voxel_value]
                color = rgba[:3]
                opacity = rgba[3]
                
                # Beer-Lambert law compositing
                # C_out = C_in + (1 - alpha_in) * C_sample * alpha_sample
                # alpha_out = alpha_in + (1 - alpha_in) * alpha_sample
                
                weight = (1.0 - accumulated_alpha) * opacity
                accumulated_color += weight * color
                accumulated_alpha += weight
                
                if accumulated_alpha > 0.999: # Early ray termination
                    break
                    
                t += view_params['step_size']
            
            alpha_plane[y, x] = accumulated_color

    return alpha_plane

def composite_alpha_plane(plane1, plane2):
    """
    Pre-multiplied alpha compositing of two 2D planes.
    """
    # [G] Standard Porter-Duff over operator
    c1, a1 = plane1[:3], plane1[3]
    c2, a2 = plane2[:3], plane2[3]
    
    new_alpha = a1 + a2 * (1 - a1)
    new_color = c1 + c2 * (1 - a1)
    
    return np.concatenate([new_color, [new_alpha]])

def srend_render_loop():
    """
    Main rendering routine called from within the simulation code.
    """
    # 1. Local Data Preparation
    # [G] User must provide simulation_data and scaling parameters
    local_brick_bytes = scale_data_to_bytes(simulation_data, min_val, max_val)
    color_table = define_color_table()
    
    # 2. Local Rendering
    local_rendering = ray_cast_brick(local_brick_bytes, color_table, view_params)
    
    # 3. Distributed Compositing
    # If this rank is a leaf in the rendering tree (no children), it sends its local rendering
    # If it is an internal node, it receives from children, composites, and sends to parent
    
    current_plane = local_rendering
    
    # Receive from children (if any)
    # [G] The rendering tree structure determines who sends to whom.
    # For simplicity, assume a binary tree or user-defined list of sources.
    for source_rank in range(num_sources):
        req = comm.Irecv(source=source_rank, tag=view_index)
        child_plane = req.wait()
        current_plane = composite_alpha_plane(current_plane, child_plane)
    
    # Send to parent (if not root)
    if target_rank != -1:
        # Send the composited plane up the tree
        # [G] MPI Send is non-blocking in Srend to minimize overhead
        comm.Isend(current_plane, dest=target_rank, tag=view_index)
    else:
        # This is the root (finisher)
        # Write image to disk
        write_image_to_disk(current_plane, output_filename)

# Simulation Loop Integration
# while simulation_not_finished:
#     update_simulation_state()
#     if render_now():
#         srend_render_loop()
```

### Parameters
*   **Data Format**: Bytes (`character*1` in Fortran). The user must scale real data to this format [S1].
*   **Color Table**: User-defined mapping from byte values to RGBA [S1].
*   **Rendering Parameters**: Passed in every call. Includes view index, camera parameters, and projection type [S1].
*   **Composer Parameters**: View index, number of sources (children in the tree), and target rank (parent in the tree) [S1].
*   **Finisher Parameters**: View index, number of sources, and output file name [S1].

### Edge Cases and Pitfalls
*   **Grid Interpolation**: If the simulation uses stretched grids or terrain-following vertical coordinates, data arrays must be interpolated to spatial arrays for Srend to maintain spatial accuracy [S1]. Failure to do so results in geometric distortion [S1].
*   **Redundant Information**: Every renderer sends identical image finishing parameters (e.g., output file name) up the tree, even though only the finisher uses them. This is a design choice to simplify usage, accepting a small data payload overhead [S1].
*   **No Load Balancing**: Srend has no supervisory mechanism. If simulation workloads are uneven, some ranks may render while others are idle, or rendering may become a bottleneck if not carefully timed [S1].
*   **Data Scaling**: The user must manually scale data to bytes and define color tables. This requires domain knowledge and is not automated [S1].

## Operating characteristics
*   **Time Complexity**: The rendering completion time (simulation halt time) depends primarily on the rendering work itself. There is no synchronization or management overhead beyond MPI messaging [S1].
*   **Memory Complexity**: Data is partitioned into bricks. Each rank only stores its local brick in memory. The 2D alpha planes are small compared to the 3D volume data [S1].
*   **Measured Performance**: Results show modest rendering overhead and fine quality imagery with high potential for scaling [S1]. The rendering work is insignificant compared to the simulation work of updating volume data according to PDEs [S1].
*   **Strengths**:
    *   Eliminates data I/O overhead by rendering in-core [S1].
    *   Simple implementation: no load balancing or supervisory mechanisms [S1].
    *   Scalable: exploits embarrassingly parallel ray casting and associative compositing [S1].
    *   Easy integration: Fortran code can be embedded via ordinary subroutine calls [S1].
*   **Limitations**:
    *   Requires user to handle data scaling and color table definition [S1].
    *   No automatic load balancing [S1].
    *   Requires Cartesian grids (or interpolated data for non-Cartesian) [S1].
    *   Redundant data transmission up the rendering tree [S1].
*   **When to use**: When visualizing CFD simulations on Cartesian grids where reducing data I/O is critical, and the simulation code is written in Fortran. Suitable for development, testing, education, and guiding post-processing [S1].
*   **When not to use**: When the simulation uses complex non-Cartesian grids without interpolation support, or when automatic load balancing is required for highly heterogeneous workloads [S1].

## Background
*   **Origin**: Srend was developed at the University of Minnesota / LCSE (Laboratory for Computational Science and Engineering) [S1].
*   **History**: It was modified from its original form used in a terrain flow code studying smoke and pollen advection in the St. Louis River region [S1]. It was redesigned to visualize data from 15,363 star simulations planned for 2014-15 [S1]. The "S" in Srend originally stood for "spherical data rendering" (lon/lat view for stellar data), but it was generalized for Cartesian grid CFD [S1].
*   **Development**: It was embedded in three major codes: Piecewise Parabolic Method (PPMstar), Cloud Model 1 (CM1), and Weather Research Forecast model (WRF) [S1].
*   **Usage**: Used for immediate, quality imagery generation during simulation runs, reducing data output and storage requirements [S1].

## Sources
[S1] In-core volume rendering for Cartesian grid fluid dynamics simulations, https://doi.org/10.1145/2792745.2792780, fetched 2026-11-05
