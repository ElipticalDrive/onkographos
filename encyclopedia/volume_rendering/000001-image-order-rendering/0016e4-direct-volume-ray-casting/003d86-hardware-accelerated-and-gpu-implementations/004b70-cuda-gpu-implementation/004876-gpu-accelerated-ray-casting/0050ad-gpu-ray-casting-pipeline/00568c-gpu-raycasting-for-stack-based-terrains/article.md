# GPU Raycasting for Stack-Based Terrains

Also called: Direct Volume Rendering of Stack-Based Terrains, GPU Raycasting for SBRT, SBRT Raycasting

## Overview
GPU Raycasting for Stack-Based Terrains (SBRT) is a direct volume rendering technique adapted for geoscientific terrain data, implemented on the GPU to enable real-time visualization [S1]. It solves the problem of efficiently rendering complex, layered subsurface structures by casting rays from a virtual camera through each output buffer pixel and composing sampled colors along the ray path [S1, S1]. What sets it apart from standard raycasting is its specific adaptation to the SBRT data structure, which uses a stack-based representation of geological intervals, and its hybrid deferred strategy for normal calculation to avoid the performance bottlenecks associated with 3D convolution on sparse stack data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** SBRT dataset (grid dimensions $grid_{ancho} \times grid_{alto}$, maximum intervals $m$), camera parameters, light source position, Transfer Function (discrete lookup table), and rendering resolution.
*   **Output:** Final color image buffer.

**Data Structures:**
The volumetric data is encoded into one or two GPU structures (textures or buffers). The specific layout depends on the chosen memory approach (see Subtopics). Common components include:
*   **Index Structure:** Maps 2D grid coordinates to stack start positions and sizes.
*   **Interval Structure:** Stores material attributes and accumulated heights for each interval.
*   **Transfer Function:** Discrete lookup table mapping material values to colors.

**Numbered Steps:**

1.  **Proxy Geometry Setup:** Use the axis-aligned bounding box (AABB) of the terrain dataset as the proxy geometry for ray intersection [S1]. This maintains simplicity and fits the terrain shape [S1].
2.  **Data Encoding:** Encode the SBRT data into the chosen GPU structure(s) (e.g., 2D textures, 3D texture, or SSBOs) [S1]. The data is organized such that the stack is extracted first, then iterated over its intervals [S1].
3.  **Ray Casting Loop:** For each pixel in the output buffer, cast a ray from the virtual camera [S1].
4.  **Ray-Box Intersection:** Intersect the ray with the bounding box to determine the entry and exit points of the ray within the terrain volume [S1].
5.  **Adaptive Ray Marching:** Traverse the SBRT stack along the ray to sample attributes and compose color [S1]:
    *   **Empty Space:** When the ray is in empty space, the step length equals the terrain cell resolution [S1].
    *   **Non-Empty Material:** Once the ray collides with non-empty material, increase the sampling frequency to avoid artifacts [S1].
    *   **Stack Traversal:** For each step, identify the current stack using the index structure, then iterate through its intervals to find the active material [S1].
6.  **Color Composition:** Map the sampled material values to colors using the discrete Transfer Function [S1]. Use front-to-back alpha blending for composition [S1].
7.  **Termination:** Stop the ray algorithm when the accumulated opacity exceeds a threshold of $1 - \epsilon$, assuming new contributions are irrelevant [S1].
8.  **Deferred Normal Calculation (Hybrid Strategy):**
    *   **Pass 1:** Render the color from the Transfer Function and store the ray-collision distance (depth) in a depth map [S1].
    *   **Pass 2:** Calculate surface normals and compose the final pixel color [S1].
        *   Obtain the 3D position of the convolution mask center by performing inverse projection from the depth map [S1].
        *   Determine if a voxel is occupied: project the inverse projection value again; if the resulting depth is less than the camera-to-inverse-projection distance, the voxel is part of the surface [S1].
        *   Calculate normals based on occupied neighbors in the mask [S1].
9.  **Lighting:** Apply the Lambertian diffuse BRDF lighting model to the final color [S1].

**Formulas:**

*   **Lambertian Diffuse BRDF:**
    $$I_p = i_a + k_d (\vec{L} \cdot \vec{N}) i_d$$
    Where:
    *   $I_p$: Final intensity at point $p$.
    *   $i_a$: Ambient light intensity.
    *   $i_d$: Diffuse light intensity.
    *   $k_d$: Lambertian reflectance.
    *   $\vec{L}$: Vector from the object to the light source.
    *   $\vec{N}$: Surface normal vector at point $p$ [S1].

*   **Super-texels Dimensions (Approach 2):**
    $$texture_{ancho} = grid_{ancho} * \lceil \sqrt{m} \rceil$$
    $$texture_{alto} = grid_{alto} * \lceil \sqrt{m} \rceil$$
    Where $\lceil \cdot \rceil$ is the ceiling operator and $m$ is the maximum number of intervals among all stacks [S1].

*   **Rectangular Optimization (Approach 2 Variant):**
    Minimize $rec_{ancho} * rec_{alto}$ subject to:
    $$rec_{ancho} + rec_{alto} \text{ is minimized (implied by context of minimizing area/overhead)}$$
    $$rec_{ancho} * rec_{alto} \ge m$$
    $$rec_{ancho} > 0, rec_{alto} > 0$$
    Where $rec_{ancho}$ and $rec_{alto}$ are the dimensions of the rectangular super-texel region [S1].

**Parameters:**
*   $\epsilon$: Threshold for opacity termination. Not given in the sources.
*   $m$: Maximum number of intervals among all stacks in the SBRT [S1].
*   SSBO Minimum Size: OpenGL implementations must support SSBOs of at least 128 MB (read/write) [S1].

**Edge Cases and Pitfalls:**
*   **Normal Calculation Bottleneck:** Using 3D convolution for normal calculation in SBRT causes a performance bottleneck because each voxel access may require locating a new stack and iterating it sequentially [S1]. The hybrid deferred strategy avoids this [S1].
*   **Memory Waste:** Approach 1 (Two 2D Textures) wastes memory because the last cells of the last row usually remain unoccupied [S1].
*   **Precision/Boundaries:** The ray intersection with the bounding box must be handled correctly to avoid sampling outside the valid data range [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global constants and data structures
# grid_ancho, grid_alto: Dimensions of the SBRT grid
# m: Maximum number of intervals
# epsilon: Opacity termination threshold
# transfer_function: Discrete lookup table (material -> color)
# index_buffer: Stores stack start index and size for each grid cell
# interval_buffer: Stores material attribute and accumulated height for each interval
# depth_map: Render target for ray-collision distance
# final_color_buffer: Render target for final image

def render_sbrt(camera, light_position, grid_ancho, grid_alto, m, epsilon):
    # Pass 1: Color and Depth
    for y in range(grid_alto):
        for x in range(grid_ancho):
            ray_origin = camera.get_origin()
            ray_direction = camera.get_ray_direction(x, y)
            
            # Intersect with bounding box
            t_min, t_max = intersect_aabb(ray_origin, ray_direction)
            if t_min > t_max:
                continue # Ray misses terrain
            
            # Adaptive ray marching
            t = t_min
            accumulated_opacity = 0.0
            color = (0.0, 0.0, 0.0, 0.0)
            
            while t < t_max and accumulated_opacity < 1.0 - epsilon:
                # Determine step size
                if is_empty_space(ray_origin + ray_direction * t):
                    step = terrain_cell_resolution
                else:
                    step = small_step_size # Increased frequency for non-empty
                
                # Sample SBRT at current position
                material, height = sample_sbrt(ray_origin + ray_direction * t, index_buffer, interval_buffer, grid_ancho, grid_alto, m)
                
                # Map to color
                sample_color = transfer_function[material]
                
                # Front-to-back alpha blending
                alpha = sample_color[3] # Assuming RGBA
                color = color + sample_color * (1.0 - accumulated_opacity)
                accumulated_opacity += alpha * (1.0 - accumulated_opacity)
                
                t += step
            
            # Store depth for normal calculation
            depth_map[x, y] = t # Ray-collision distance
            
            # Store intermediate color (or final if no normals needed, but here we do deferred)
            # Note: The source says "obtenemos el color devuelto por la FT" in pass 1
            # and "componemos el color final" in pass 2. 
            # We store the unlit color for pass 2.
            final_color_buffer[x, y] = color

    # Pass 2: Normal Calculation and Final Lighting
    for y in range(grid_alto):
        for x in range(grid_ancho):
            # Get depth from pass 1
            depth = depth_map[x, y]
            
            # Inverse projection to get 3D position
            pos_3d = inverse_project(camera, x, y, depth)
            
            # Calculate normal using hybrid deferred strategy
            # Check neighbors in a mask around pos_3d
            normal = calculate_normal_hybrid(pos_3d, index_buffer, interval_buffer, grid_ancho, grid_alto, m)
            
            # Get unlit color from pass 1
            unlit_color = final_color_buffer[x, y]
            
            # Apply Lambertian lighting
            L = normalize(light_position - pos_3d)
            N = normalize(normal)
            kd = 1.0 # Example reflectance
            ia = 0.2 # Ambient intensity
            id = 0.8 # Diffuse intensity
            
            diffuse = kd * dot(L, N) * id
            final_intensity = ia + diffuse
            
            # Apply lighting to color
            final_color = unlit_color * final_intensity
            
            # Write to final buffer
            write_to_output_buffer(x, y, final_color)

def sample_sbrt(position, index_buffer, interval_buffer, grid_ancho, grid_alto, m):
    # Project position to grid coordinates
    gx, gz = project_to_grid(position)
    
    # Get stack index and size from index buffer
    stack_start = index_buffer[gx, gz].R
    stack_size = index_buffer[gx, gz].G
    
    # Iterate through intervals in interval buffer
    # Interval buffer stores [material (R), accumulated_height (G)]
    current_height = 0.0
    for i in range(stack_size):
        interval_idx = stack_start + i
        material = interval_buffer[interval_idx].R
        interval_height = interval_buffer[interval_idx].G
        
        # Check if position is within this interval
        # Note: The source says "comparar esta altura con G2"
        # Assuming G2 is the accumulated height of the current interval
        # and we need to check if the ray's height relative to the stack start is within this interval
        # This is a simplified interpretation; exact logic depends on SBRT height encoding
        if current_height <= position.y < current_height + interval_height:
            return material, interval_height
        
        current_height += interval_height
    
    return EMPTY_MATERIAL, 0.0

def calculate_normal_hybrid(pos_3d, index_buffer, interval_buffer, grid_ancho, grid_alto, m):
    # Define mask size (e.g., 3x3x3)
    mask_size = 1
    normal = (0.0, 0.0, 0.0)
    
    # Check neighbors in the mask
    for dx in range(-mask_size, mask_size + 1):
        for dy in range(-mask_size, mask_size + 1):
            for dz in range(-mask_size, mask_size + 1):
                neighbor_pos = pos_3d + (dx, dy, dz)
                
                # Check if neighbor is occupied
                # "Un vóxel se considera ocupado si no está ocluido por el modelo real"
                # "si al proyectar de nuevo el valor obtenido por la proyección inversa, 
                # se obtiene un valor de profundidad menor que la distancia de la cámara 
                # a la posición de la proyección inversa"
                # This implies checking if the neighbor's depth in the depth map is valid
                # and corresponds to a surface voxel
                
                # Simplified: Check if the neighbor is part of the surface
                if is_surface_voxel(neighbor_pos, index_buffer, interval_buffer, grid_ancho, grid_alto, m):
                    # Add vector to normal
                    normal += normalize(neighbor_pos - pos_3d)
    
    return normalize(normal)

def is_surface_voxel(pos, index_buffer, interval_buffer, grid_ancho, grid_alto, m):
    # Check if voxel is occupied and not occluded
    # This requires checking the SBRT data at pos
    material, _ = sample_sbrt(pos, index_buffer, interval_buffer, grid_ancho, grid_alto, m)
    if material == EMPTY_MATERIAL:
        return False
    
    # Check occlusion: project pos again and compare depths
    # This is a simplified check; actual implementation requires access to depth map
    # and camera parameters
    return True # Placeholder for complex occlusion check
```

## Subtopics

### Approach 1 (Two 2D Textures)
Uses an Index Texture ($grid_{ancho} \times grid_{alto}$) storing stack start (R) and size (G), and an Interval Texture storing material (R) and accumulated height (G). Stacks are stored row-order in the Interval Texture. *Sets apart:* Simple indexing but wastes memory in the last row. *Choose when:* Memory is not a critical constraint and simplicity is preferred. *Link:* [GPU Raycasting for Stack-Based Terrains](14281-gpu-ray-casting-pipeline.md)

### Approach 2 (Super-texels)
Uses a single 2D Texture with dimensions $grid_{ancho} * \lceil \sqrt{m} \rceil \times grid_{alto} * \lceil \sqrt{m} \rceil$. Regions of size $\lceil \sqrt{m} \rceil \times \lceil \sqrt{m} \rceil$ act as super-texels. *Sets apart:* Pseudo-indexing via regions. *Choose when:* $m$ is small and square regions are efficient. *Link:* [GPU Raycasting for Stack-Based Terrains](14281-gpu-ray-casting-pipeline.md)

### Approach 3 (2D Texture Rectangular)
Uses a 2D Texture of dimensions $(grid_{ancho} * m) \times grid_{alto}$. Intervals stored along x-axis. *Sets apart:* Checks spatial coherence against 3D textures. *Choose when:* Spatial coherence is a priority. *Link:* [GPU Raycasting for Stack-Based Terrains](14281-gpu-ray-casting-pipeline.md)

### Approach 4 (3D Texture)
Uses a 3D Texture of dimensions $grid_{ancho} \times grid_{alto} \times m$. Intervals stored along z-axis. *Sets apart:* Standard 3D texture access. *Choose when:* 3D texture hardware is efficient. *Link:* [GPU Raycasting for Stack-Based Terrains](14281-gpu-ray-casting-pipeline.md)

### Approach 5 (SSBOs)
Uses two Shader Storage Buffer Objects (SSBOs) for indices and intervals, using OpenGL std430 layout. *Sets apart:* Flexible array sizes, large memory support (128 MB min). *Choose when:* Large datasets or dynamic stack sizes. *Link:* [GPU Raycasting for Stack-Based Terrains](14281-gpu-ray-casting-pipeline.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays (pixels) and the average stack depth. Adaptive marching reduces steps in empty space. Normal calculation is deferred and depends on output buffer size, not data size [S1].
*   **Memory Complexity:** Depends on the chosen approach. Approach 1 wastes memory in the last row [S1]. Approach 5 requires SSBOs of at least 128 MB [S1].
*   **Measured Performance:** Enables real-time rendering and visual operations such as cross-sections, material layer attenuation, and borehole visualization [S1].
*   **Scaling:** Scales with GPU hardware capabilities. Avoids CPU-GPU data transfer bottleneck [S1].
*   **Strengths:** Real-time performance, efficient memory usage with SSBOs, avoids 3D convolution bottleneck for normals [S1].
*   **Limitations:** Normal calculation is limited in number compared to gradient-based methods, though sufficient for scientific applications [S1]. Approach 1 wastes memory [S1].
*   **When to use:** Geoscientific visualization requiring real-time interaction with layered terrain data [S1].
*   **When not:** When high-fidelity, gradient-based normals are critical and performance is not a constraint [S1].

## Background
*   **Origin:** Proposed by A. Graciano, A.J. Rueda & F.R. Feito in 2017 [S1].
*   **History:** Developed to address the bottleneck of data transfer between CPU and GPU in visualization systems and to enable real-time operations on SBRT data [S1].
*   **Usage:** Geoscientific applications, terrain visualization, subsurface structure visualization [S1].

## Sources
[S1] Direct Volume Rendering of Stack-Based Terrains, https://doi.org/10.2312/ceig.20171207, fetched 2026-10-26
