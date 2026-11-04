# Parallel Volume Ray Casting (PVR) Library Interface

Also called: Direct Volume Rendering for Parallel CFD, PVR library API, Parallel Volume Rendering (PVR) Algorithm

## Overview
The Parallel Volume Ray Casting (PVR) Library Interface is a software design for distributed-memory systems that enables in-place rendering of rectilinear grid data without transferring large datasets to a host computer [S1, unofficial]. It separates the rendering pipeline into local resampling (ray casting) and global compositing, allowing interactive monitoring of parallel CFD simulations [S1, unofficial]. Unlike sibling methods that may rely on binary-swap compositing or require regular data partitioning, this interface selects direct compositing to remain independent of the specific data partitioning scheme, prioritizing generic applicability over peak compositing performance [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  **Subvolume Pointer:** A pointer to the local chunk of the 3D scalar dataset assigned to the current process [S1, unofficial].
2.  **Geometry Record:** A data structure describing the spatial extent and dimensions of the local subvolume [S1, unofficial].
3.  **Rendering Specifications:** Parameters including viewing and lighting parameters, image resolution, rendering rate, and transfer functions [S1, unofficial].

**Outputs:**
1.  **Partial Image Pixels:** The rendered color and opacity values for the subset of image pixels assigned to the current process [S1, unofficial].
2.  **Final Composite Image:** The complete image formed after global compositing (typically handled by a host or dedicated node) [S1, unofficial].

### Data Structures
The library interface expects the following structures to be passed from the application program to the renderer [S1, unofficial]:

1.  **Subvolume Data:** A contiguous array of voxel values. The size depends on the data resolution (1 to 12 bytes per voxel) [S1, unofficial].
2.  **Geometry Record:** Contains dimensions of the local block $(D_x, D_y, D_z)$ and its spatial offset relative to the global domain [S1, unofficial].
3.  **Rendering Spec:** Contains camera position, view direction, image plane resolution $(N_x, N_y)$, and transfer function mappings [S1, unofficial].

### Algorithm Steps

1.  **Initialization and Broadcast:**
    The host program reads the model and rendering specifications and broadcasts them to every node processor [S1, unofficial]. The application program passes the renderer the subvolume pointer, geometry record, and rendering specifications [S1, unofficial].

2.  **Data Distribution (Pre-processing):**
    The 3D domain is decomposed into blocks (preferred over slabs) to ensure sizes in all dimensions are as close as possible [S1, unofficial]. This reduces communication costs for the underlying CFD simulation and makes rendering less view-dependent [S1, unofficial]. Each data subset must contain spatially consecutive voxels [S1, unofficial].

3.  **Boundary Replication:**
    Voxels at the boundaries of the distributed data blocks are replicated to handle interpolation near edges [S1, unofficial]. This is required because most CFD codes use a finite-difference formulation, which inherently requires neighbor data [S1, unofficial].
    *   *Note:* This step could be removed if special boundary care were implemented, but replication is the standard approach for this library [S1, unofficial].

4.  **Local Resampling:**
    On each processor, rays are cast from the eye through the image plane into the local data volume [S1, unofficial].
    *   **Sampling Rate:** One to two samples per voxel are computed [S1, unofficial].
    *   **Interpolation:** Color and opacity are computed by interpolating from data values using trilinear interpolation for parallelepiped elements (8 vertices) [S1, unofficial].
    *   *Constraint:* No communication is required during this phase [S1, unofficial].

5.  **Local Compositing:**
    The color and opacity of samples along each ray segment within the local block are composited front-to-back [S1, unofficial]. The compositing operation is associative, allowing independent processing of segments [S1, unofficial].

6.  **Global Compositing (Direct Method):**
    *   **Image Subdivision:** The image plane is subdivided, and subsets of image pixels are assigned to each node [S1, unofficial].
    *   **Direct Transmission:** Each node sends its rendered partial image pixels directly to the node assigned that specific portion of the image plane [S1, unofficial].
    *   **Accumulation and Final Composite:** Nodes accumulate these incoming partial image pixels in an array. After all rendering is complete, they composite them in proper depth order to form the final image [S1, unofficial].
    *   *Selection Rationale:* Direct compositing is selected over binary-swap compositing because binary-swap requires regular data partitioning and special data structures, making it less desirable for a generic library design [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pvr_library_interface(subvolume_ptr, geometry_record, rendering_specs):
    """
    Main entry point for the PVR library.
    
    Args:
        subvolume_ptr: Pointer to local 3D array of voxel values.
        geometry_record: Object containing dimensions (dx, dy, dz) and global offset.
        rendering_specs: Object containing camera params, image resolution (nx, ny), 
                         transfer functions, etc.
    """
    
    # 1. Local Resampling
    # Cast rays from eye through image plane into local volume
    # Image resolution is defined in rendering_specs
    image_width = rendering_specs.image_resolution.x
    image_height = rendering_specs.image_resolution.y
    
    # Determine which pixels this process is responsible for
    # In Direct Compositing, pixels are assigned to nodes.
    # This process renders only its assigned subset of pixels.
    assigned_pixels = get_assigned_pixel_subset(image_width, image_height)
    
    local_partial_image = []
    
    for pixel in assigned_pixels:
        ray = create_ray_from_eye(pixel, rendering_specs.camera)
        
        # 2. Sample the volume along the ray
        # Sample rate: 1-2 samples per voxel
        samples = sample_ray(ray, subvolume_ptr, geometry_record, 
                             sample_rate=1.5, 
                             interpolation='trilinear')
        
        # 3. Local Compositing (Front-to-Back)
        # Compute color and opacity for each sample
        composited_color = 0.0
        composited_opacity = 0.0
        
        for sample in samples:
            # Interpolate value from subvolume using trilinear interpolation
            # at the sample's 3D coordinate
            value = trilinear_interpolate(subvolume_ptr, geometry_record, sample.position)
            
            # Map value to color and opacity using transfer function
            color, opacity = apply_transfer_function(value, rendering_specs.transfer_function)
            
            # Front-to-back compositing
            # C_out = C_in + (1 - A_in) * C_sample * A_sample
            # A_out = A_in + (1 - A_in) * A_sample
            composited_color = composited_color + (1 - composited_opacity) * color * opacity
            composited_opacity = composited_opacity + (1 - composited_opacity) * opacity
            
            # Early ray termination if opaque enough (optional optimization)
            if composited_opacity > 0.99:
                break
        
        local_partial_image.append({
            'pixel_id': pixel.id,
            'color': composited_color,
            'opacity': composited_opacity
        })
        
    # 4. Global Compositing (Direct Method)
    # Send partial image pixels to the node responsible for those pixel locations
    target_node = get_node_for_pixel_subset(assigned_pixels)
    send_pixels_to_node(target_node, local_partial_image)
    
    # 5. Accumulate incoming pixels from other nodes
    # This process also receives pixels for its own assigned pixel subset
    incoming_pixels = receive_pixels_from_other_nodes()
    
    # Combine local and incoming pixels for the assigned subset
    final_pixel_set = local_partial_image + incoming_pixels
    
    # Composite in proper depth order
    # Note: In direct compositing, the "depth order" refers to the order 
    # of ray segments from different blocks. The library assumes the 
    # transmission order or a specific merging strategy handles this.
    # The source states: "composite them in proper order after all rendering is completed"
    final_image_for_this_node = composite_final_image(final_pixel_set)
    
    return final_image_for_this_node

def trilinear_interpolate(data_ptr, geometry, position):
    """
    Performs trilinear interpolation on the local subvolume.
    Requires boundary replication to be handled by the data structure 
    or pre-processing step.
    """
    # Map global position to local coordinates
    local_pos = position - geometry.offset
    
    # Check bounds (should be within replicated boundaries)
    if not is_within_bounds(local_pos, geometry.dimensions):
        return 0.0 # Or handle edge case
        
    # Extract 8 corner values
    # ... implementation details of trilinear interpolation ...
    v000 = data_ptr[local_pos.x, local_pos.y, local_pos.z]
    v100 = data_ptr[local_pos.x+1, local_pos.y, local_pos.z]
    # ... other 6 vertices ...
    
    # Interpolate
    # ... standard trilinear formula ...
    return interpolated_value

def get_assigned_pixel_subset(width, height):
    """
    Determines which pixels this process is responsible for.
    In direct compositing, the image plane is subdivided.
    """
    # Implementation depends on the specific image subdivision strategy
    # (e.g., interleaved fashion for load balancing)
    pass

def get_node_for_pixel_subset(pixels):
    """
    Determines which node is responsible for a given set of pixels.
    """
    pass

def send_pixels_to_node(node_id, pixels):
    """
    Sends pixel data to the target node via MPI or native communication library.
    """
    pass

def receive_pixels_from_other_nodes():
    """
    Receives pixel data from other nodes.
    """
    pass

def composite_final_image(pixels):
    """
    Combines pixels from different blocks in proper depth order.
    """
    pass
```

### Parameters
*   **Sample Rate:** 1 to 2 samples per voxel [S1, unofficial].
*   **Interpolation:** Trilinear interpolation for parallelepiped elements [S1, unofficial].
*   **Data Type:** Voxels can take 1 to 12 bytes depending on resolution and algorithm [S1, unofficial].

### Edge Cases and Pitfalls
*   **Load Imbalance:** Imbalanced load can degrade overall performance by 20-40% [S1, unofficial]. Dynamic load balancing is challenging in runtime visualization because simulation and visualization steps are separated, making static preprocessing impractical [S1, unofficial].
*   **Boundary Interpolation:** Without boundary replication, trilinear interpolation near block edges will fail or produce artifacts. Replication is required for finite-difference CFD codes [S1, unofficial].
*   **Memory Constraints:** The renderer requires additional memory for data, parameters, and partial images. The memory space is dominated by the volume data size, final sub-image size, and partial image size [S1, unofficial].
*   **Display Bottleneck:** If a host processor collects and combines subimages, the serial data stream limits display rates [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory Space:** The memory space for the renderer is calculated as:
    $$ \text{Memory} = n_{voxel} + 4 \times \left(\frac{n}{p} + \frac{n}{p^{1/2}}\right) \text{ bytes} $$
    Where:
    *   $n_{voxel}$ is the total number of voxels [S1, unofficial].
    *   $p$ is the number of processors [S1, unofficial].
    *   $n$ is the number of pixels in the final image [S1, unofficial].
    *   The term $4 \times ((n/p) + (n/p^{1/2}))$ accounts for the final sub-image size and partial image size [S1, unofficial].
    *   Each voxel can take from 1 to 12 bytes [S1, unofficial].

*   **Communication Cost:** For parallel compositing on a bidirectional torus, the communication cost is:
    $$ C_1 \times p \times n \times t_{trans} + C_2 \times t_{overhead} $$
    Where:
    *   $C_1, C_2$ are positive constants smaller than 1 [S1, unofficial].
    *   $t_{trans}$ is the per-pixel transfer cost [S1, unofficial].
    *   $t_{overhead}$ is the per-message overhead [S1, unofficial].
    *   The analysis shows that communication cost decreases as the number of processors ($p$) increases, making the direct compositing method scalable [S1, unofficial].

### Performance and Scaling
*   **Scalability:** The rendering phase (resampling) has no communication and scales well. The compositing phase involves inter-processor communication but scales acceptably due to decreasing communication cost with more processors [S1, unofficial].
*   **Performance Degradation:** Imbalanced load degrades performance by 20-40%. Efficient dynamic load balancing could remove at least 50% of this degradation [S1, unofficial].
*   **Hardware:** Tested on Intel Paragon XP/S using 8 to 216 processors [S1, unofficial]. Implemented in C/C++ with Intel's native communication library NX, with potential for MPI/PVM [S1, unofficial].

### Strengths and Limitations
*   **Strengths:**
    *   Allows in-place rendering without transferring large datasets to a host [S1, unofficial].
    *   Direct compositing is independent of data partitioning, making it generic [S1, unofficial].
    *   Interactive visual response is helpful for monitoring CFD simulations [S1, unofficial].
*   **Limitations:**
    *   Dynamic load balancing is non-trivial and not fully implemented in the base design [S1, unofficial].
    *   Rectilinear grids are assumed; extension to curvilinear or unstructured grids requires additional algorithms [S1, unofficial].
    *   Display rates can be limited by serial data streams if a host processor handles final combination [S1, unofficial].

## Background

### Origin and History
The Parallel Volume Ray Casting (PVR) Library Interface was designed for distributed memory message passing environments to support runtime visualization of parallel CFD simulations [S1, unofficial]. It was developed to allow interactive monitoring of simulations, such as a three-dimensional Navier-Stokes solver, without the overhead of transferring data to a host computer [S1, unofficial].

### Development
The library was tested on the Intel Paragon XP/S system with 8 to 216 processors [S1, unofficial]. The implementation used C/C++ and Intel's native communication library NX [S1, unofficial]. The design focuses on a simple API where the application passes a pointer to the subvolume, a geometry record, and rendering specifications [S1, unofficial].

### Usage
It is primarily used in scientific visualization for parallel CFD, specifically for visualizing flowfields from Navier-Stokes solvers [S1, unofficial]. The library is designed for rectilinear grids, which are widely used in CFD, though it can be extended to other grid types [S1, unofficial].

## Sources

[S1] DTIC ADA302573: Runtime Volume Visualization for Parallel CFD., https://archive.org/details/DTIC_ADA302573, fetched 2026-10-26 (unofficial)
