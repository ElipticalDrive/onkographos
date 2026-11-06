# Volume Ray Casting with Barycentric Interpolation and Phong Lighting

Also called: CPU Volume Ray Caster, GPU Volume Ray Caster, Ray Tracing Volume Rendering

## Overview
A volume rendering technique that casts rays from a viewing plane through a 3D voxel grid, sampling voxel values using barycentric interpolation within the unit cube, and accumulating lighting contributions (ambient, diffuse, specular) along the ray path [S1, unofficial]. It projects values to a view plane to produce a 2D image by casting parallel rays from the eye to voxels [S2, unofficial]. It is distinguished by the explicit use of barycentric interpolation for spatial sampling and the Phong shading model for lighting calculation, rather than simple nearest-neighbor or trilinear interpolation combined with standard transfer function opacity alone.

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D voxel grid (scalar data), viewing parameters (camera position, view matrix, canvas dimensions), light position, light color, material properties (ambient, diffuse, specular coefficients, shininess), transfer function (mapping scalar data to opacity), sample period (step size).
*   **Outputs:** 2D image (color buffer).

**Data Structures:**
*   **Voxel Grid:** A 3D array of scalar values. The exact layout (e.g., row-major, column-major) is not specified in the sources [S1, S2, unofficial].
*   **Viewing Plane:** Defined by a canvas width and height, and a center point. In the reference implementation, `CANVAS_WIDTH = 400`, `CANVAS_HEIGHT = 400`, `CENTER_X = 200`, `CENTER_Y = 200` [S1, unofficial].
*   **Transfer Function:** Maps scalar data to opacity. Opacity ranges from 0 (transparent) to 1 (opaque) [S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize the viewing plane and volume data structures. Set up vertex arrays for canvas points and shader storage buffers for volume data [S1, unofficial]. Convert each data element into a voxel [S2, unofficial].
2.  **Ray Generation:** For each pixel on the canvas, compute the ray origin from the viewing plane and the ray direction from the view matrix [S1, unofficial].
3.  **Classification:** Classify each voxel by assigning opacity using lookup tables and transfer functions that map scalar data to opacity [S2, unofficial].
4.  **Ray Marching and Accumulation:** Iterate along the ray from $t=0$ to $t=\text{distance}$ with a step size of `sample_period` [S1, unofficial].
    *   **Sampling:** At the current position along the ray, sample the voxel value using barycentric interpolation within the unit cube [S1, unofficial].
    *   **Lighting:** Calculate lighting contributions using the standard Phong shading equation [S2, unofficial].
        *   Calculate a gradient vector for the current voxel to determine the surface normal [S2, unofficial].
        *   Compute the reflection direction based on the normal and light direction.
        *   Calculate ambient, diffuse, and specular components.
    *   **Accumulation:** Accumulate the lighting result along the ray path.
5.  **Output:** Clamp the accumulated value, scaled by a dimming factor, to the range $[0, 1]$ and output as color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_ray_casting_with_barycentric_phong(voxel_grid, view_matrix, light_pos, light_color, material, transfer_func, sample_period, canvas_width, canvas_height):
    # Initialize output image
    image = zeros((canvas_height, canvas_width, 3))
    
    # Precompute inverse view matrix if needed for ray direction calculation
    # Note: Ray direction is derived from view matrix and pixel coordinates
    
    for y in range(canvas_height):
        for x in range(canvas_width):
            # 1. Compute ray origin and direction from viewing plane
            # Ray origin is typically the camera position
            ray_origin = compute_ray_origin(view_matrix, x, y, canvas_width, canvas_height)
            ray_direction = compute_ray_direction(view_matrix, x, y, canvas_width, canvas_height)
            
            accumulated_color = vec3(0.0, 0.0, 0.0)
            accumulated_opacity = 0.0
            
            # 2. Iterate along the ray
            t = 0.0
            while t < distance_to_volume(ray_origin, ray_direction):
                # 3. Sample voxel using barycentric interpolation
                # Determine the unit cube containing the current point
                voxel_pos = ray_origin + t * ray_direction
                voxel_value = barycentric_interpolate(voxel_grid, voxel_pos)
                
                # 4. Classify: Map scalar value to opacity
                opacity = transfer_func(voxel_value)
                
                # 5. Calculate Lighting (Phong)
                # Calculate gradient for normal estimation
                normal = compute_gradient_normal(voxel_grid, voxel_pos)
                
                # Phong shading components
                ambient = material.ambient * light_color
                diffuse = material.diffuse * light_color * max(dot(normal, normalize(light_pos - voxel_pos)), 0.0)
                specular = material.specular * light_color * pow(max(dot(reflect(normalize(light_pos - voxel_pos), normal), normalize(-ray_direction)), 0.0), material.shininess)
                
                voxel_color = ambient + diffuse + specular
                
                # 6. Accumulate (Front-to-Back)
                # Note: Standard volume rendering compositing is alpha blending
                # C = C_prev + (1 - A_prev) * C_voxel * opacity_voxel
                # A = A_prev + (1 - A_prev) * opacity_voxel
                
                # However, the source mentions "accumulating lighting contributions" and "clamping accumulated value scaled by a dimming factor"
                # This suggests a simpler accumulation or a specific implementation detail.
                # We will follow the standard front-to-back compositing for volume rendering unless specified otherwise.
                # The source [S1, unofficial] mentions "accumulated value scaled by a dimming factor", which might refer to the final output scaling.
                
                contribution = voxel_color * opacity
                accumulated_color = accumulated_color + (1.0 - accumulated_opacity) * contribution
                accumulated_opacity = accumulated_opacity + (1.0 - accumulated_opacity) * opacity
                
                # 7. Step forward
                t += sample_period
                
            # 8. Output
            # Clamp and scale by dimming factor
            final_color = clamp(accumulated_color * material.dimming_factor, 0.0, 1.0)
            image[y, x] = final_color
            
    return image

def barycentric_interpolate(voxel_grid, point):
    # Find the unit cube containing the point
    # Let the point be (x, y, z)
    # The cube is defined by integer coordinates (ix, iy, iz) such that:
    # ix <= x < ix+1, iy <= y < iy+1, iz <= z < iz+1
    
    ix = floor(point.x)
    iy = floor(point.y)
    iz = floor(point.z)
    
    # Barycentric coordinates within the unit cube
    # For a cube, this is equivalent to trilinear interpolation weights
    # However, barycentric interpolation is typically defined for simplices (tetrahedra).
    # The source specifies "barycentric interpolation within the unit cube".
    # This likely implies decomposing the cube into tetrahedra and using barycentric coordinates,
    # or it is a misnomer for trilinear interpolation.
    # Given the ambiguity, we assume trilinear interpolation as it is standard for cubes.
    # If true barycentric interpolation on tetrahedra is required, the cube must be split.
    
    # Trilinear interpolation weights
    wx = point.x - ix
    wy = point.y - iy
    wz = point.z - iz
    
    # Get the 8 corner values
    v000 = voxel_grid[ix, iy, iz]
    v100 = voxel_grid[ix+1, iy, iz]
    v010 = voxel_grid[ix, iy+1, iz]
    v110 = voxel_grid[ix+1, iy+1, iz]
    v001 = voxel_grid[ix, iy, iz+1]
    v101 = voxel_grid[ix+1, iy, iz+1]
    v011 = voxel_grid[ix, iy+1, iz+1]
    v111 = voxel_grid[ix+1, iy+1, iz+1]
    
    # Interpolate
    c00 = lerp(v000, v100, wx)
    c10 = lerp(v010, v110, wx)
    c01 = lerp(v001, v101, wx)
    c11 = lerp(v011, v111, wx)
    
    b0 = lerp(c00, c10, wy)
    b1 = lerp(c01, c11, wy)
    
    return lerp(b0, b1, wz)

def lerp(a, b, t):
    return a + t * (b - a)

def compute_gradient_normal(voxel_grid, point):
    # Calculate gradient vector for each voxel
    # This is typically done using finite differences
    dx = voxel_grid[point.x+1, point.y, point.z] - voxel_grid[point.x-1, point.y, point.z]
    dy = voxel_grid[point.x, point.y+1, point.z] - voxel_grid[point.x, point.y-1, point.z]
    dz = voxel_grid[point.x, point.y, point.z+1] - voxel_grid[point.x, point.y, point.z-1]
    return normalize(vec3(dx, dy, dz))
```

**Parameters:**
*   `sample_period`: Step size for ray marching. Typical values in the source are 64, 128, 256, 512 sample points [S1, unofficial].
*   `dimming_factor`: A scaling factor applied to the accumulated value before clamping [S1, unofficial]. Not given in the sources.
*   `material`: Contains ambient, diffuse, specular coefficients, shininess, and dimming factor [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The gradient calculation requires neighboring voxels. Boundary conditions (e.g., zero-padding, clamping) must be defined [S2, unofficial].
*   **Precision:** Floating-point precision may affect the accuracy of barycentric interpolation and gradient calculation [S1, S2, unofficial].
*   **Degenerate Input:** Empty volumes or volumes with constant scalar values may result in no visible structure [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of sample points per ray [S1, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the number of voxels in the grid [S2, unofficial].
*   **Measured Performance:** The source provides speedup data for different numbers of sample points (64, 128, 256, 512) and processors, but specific hardware details and absolute performance metrics are not provided [S1, unofficial].
*   **Strengths:** Parallelizable across processors without communication overhead [S2, unofficial]. Provides smooth interpolation and realistic lighting.
*   **Limitations:** Computationally expensive due to per-pixel ray marching and lighting calculations. Performance degrades with higher sample counts [S1, unofficial].
*   **When to Use:** When high-quality lighting and smooth interpolation are required, and computational resources are sufficient.
*   **When Not to Use:** When real-time performance is critical and hardware resources are limited.

## Background
The method is based on volume ray casting, a technique for visualizing 3D scalar fields [S1, unofficial]. The specific implementation using barycentric interpolation and Phong lighting is described in the context of a project for learning volume ray casting [S1, unofficial]. The use of VolPack library functions for classification and shading is mentioned in the context of temporal rendering and parallel processing [S2, unofficial]. The technique is applicable to time-varying data visualization [S2, unofficial].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
