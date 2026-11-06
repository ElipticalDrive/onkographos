# Mixed Geometry Rendering

Also called: Direct Volume Rendering with Z-Buffer Geometry, Wireframe and Surface Combination

## Overview
Mixed Geometry Rendering combines direct volume ray casting with explicit geometric primitives (such as wireframes or surfaces) to provide structural context to volumetric data [S1]. It solves the problem of depth perception and structural clarity in scalar fields by rendering the volume's internal density while simultaneously displaying an external or internal geometric skeleton [S1]. Unlike pure volume rendering, which relies solely on shading and opacity for shape cues, this method uses a standard Z-buffer to render geometry that occludes the volume behind it, while the volume correctly transmits light from geometry rendered in front [S1].

## How it works
The method integrates two rendering passes: a geometric pass for explicit structures and a ray-casting pass for the scalar volume.

**Inputs and Outputs**
*   **Inputs:** A scalar volume dataset (grid or unstructured), a set of geometric primitives (e.g., wireframe edges, surface meshes), a camera position $\mathbf{o}$ and direction $\mathbf{d}$, a transfer function $T$, and a lighting model.
*   **Outputs:** A final 2D image where volume density and geometric structure are composited.

**Data Structures**
*   **Volume Grid:** A 3D array of scalar values. For unstructured data (e.g., tetrahedral cells), the memory layout stores node coordinates and cell connectivity [S1].
*   **Geometry Buffer:** A standard Z-buffer (depth buffer) storing the depth $z$ of the closest geometric primitive for each pixel [S1].
*   **Pixel Color Buffer:** Stores the accumulated color $C_{pixel}$ for each pixel [S1].

**Numbered Steps**

1.  **Initialize Buffers:** Clear the Z-buffer to infinity and the Color buffer to black (or background color) [S1].
2.  **Render Geometry:** Render the explicit geometric primitives (e.g., wireframe) into the Z-buffer using standard rasterization or ray-triangle intersection [S1]. This establishes the depth of the structural boundaries [S1].
3.  **Compute Gradients:** For each voxel in the volume, compute the gradient vector $\nabla V$ to serve as a surface normal for lighting [S1].
    *   The gradient is computed using differences of values and coordinates between three pairs of neighboring points [S1].
    *   The X component of the gradient is calculated as the ratio of two determinants [S1]:
        $$
        \nabla V_x = \frac{\det \begin{pmatrix} A_{f0} & A_{y0} & A_{z0} \\ A_{f1} & A_{y1} & A_{z1} \\ A_{f2} & A_{y2} & A_{z2} \end{pmatrix}}{\det \begin{pmatrix} A_{x0} & A_{y0} & A_{z0} \\ A_{x1} & A_{y1} & A_{z1} \\ A_{x2} & A_{y2} & A_{z2} \end{pmatrix}}
        $$
        where $A$ represents differences in values ($A_f$) and coordinates ($A_x, A_y, A_z$) [S1].
    *   The Y and Z components are computed similarly by substituting value differences into the second and third columns of the numerator determinant, respectively [S1].
4.  **Ray Casting and Compositing:** For each pixel, cast a ray through the volume [S1].
    *   Determine the ray segment $[t_{start}, t_{end}]$ within the volume domain [S1].
    *   Discretize the ray into samples [S1].
    *   For each sample point $\mathbf{p}_i$:
        a.  **Classification:** Map the scalar value $V(\mathbf{p}_i)$ to color $C_i$ and opacity $\alpha_i$ using the transfer function [S1].
        b.  **Shading:** Compute the lighting contribution using the pre-computed gradient $\nabla V$ as the normal and a simple lighting equation (e.g., diffuse reflection) [S1].
        c.  **Integration:** Accumulate the color and opacity along the ray [S1].
5.  **Final Composition:** After integrating the volume along the ray, multiply the stored color in the pixel (from the geometry pass) by the transmittance of the volume $(1 - \text{integral of opacity along the ray})$ and add this product to the ray's accumulated color [S1].
    *   Formula:
        $$
        C_{final} = C_{geometry} \cdot (1 - \int \alpha \, dt) + C_{volume}
        $$
        where $C_{geometry}$ is the color from the Z-buffer geometry pass [S1].

**Parameters**
*   **Resolution:** Performance results cited use a 512x512 pixel window [S1].
*   **Lighting Model:** Simple lighting equation using gradient normals [S1].

**Edge Cases and Pitfalls**
*   **Occlusion Artifacts:** Wireframe geometry completely occludes volume data behind it. If the wireframe is dense, it may hide important internal volume structures [S1].
*   **Transparency:** Volume data in front of the wireframe correctly transmits colors from the geometry, but the geometry does not transmit through the volume [S1].
*   **Memory Usage:** High memory usage for cell-organized (unstructured) data. For example, a model with 381,548 tetrahedral cells required over 50 Megabytes [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mixed_geometry_render(volume, geometry, camera, transfer_function, lighting_model):
    # 1. Initialize buffers
    z_buffer = array(width, height, init=INFINITY)
    color_buffer = array(width, height, init=(0, 0, 0))
    
    # 2. Render Geometry into Z-buffer
    # Standard rasterization or ray-triangle intersection
    for prim in geometry:
        render_primitive(prim, z_buffer, color_buffer)
        
    # 3. Pre-compute gradients for volume
    gradients = compute_gradients(volume)
    
    # 4. Ray Casting for Volume
    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = camera.get_ray(x, y)
            
            # Determine volume bounds
            t_start, t_end = intersect_volume(ray_origin, ray_direction, volume.bounds)
            
            accumulated_volume_color = (0, 0, 0)
            total_opacity = 0.0
            
            # Discretize ray
            num_samples = calculate_samples(t_start, t_end, resolution)
            for i in range(num_samples):
                t = t_start + i * (t_end - t_start) / num_samples
                point = ray_origin + t * ray_direction
                
                # Sample volume
                scalar_value = sample_volume(volume, point)
                
                # Classification
                color, opacity = transfer_function(scalar_value)
                
                # Shading using gradient
                normal = gradients[point]
                light_color = lighting_model(normal, ray_direction)
                shaded_color = color * light_color
                
                # Compositing (Front-to-Back)
                # Note: The specific integration formula in [S1] implies 
                # adding volume color weighted by transmittance, 
                # but standard over-operator is implied for volume accumulation.
                # [S1] states: "integrate these values along the ray"
                # and "Multiply the stored color... by (1 - integral of opacity)"
                
                # Accumulate volume contribution
                # Using standard front-to-back for volume part
                transmittance = 1.0 - total_opacity
                accumulated_volume_color += transmittance * opacity * shaded_color
                total_opacity += transmittance * opacity
                
                if total_opacity >= 1.0:
                    break
            
            # 5. Final Composition
            # Geometry color is already in color_buffer[x, y]
            geo_color = color_buffer[x, y]
            
            # Volume transmittance
            vol_transmittance = 1.0 - total_opacity
            
            # Combine: Geometry is occluded by volume? 
            # [S1] says: "wireframe geometry completely occludes volume data behind it"
            # and "volume data in front ... correctly transmits colors from the geometry"
            # This implies the geometry is rendered first (background), 
            # and volume is rendered on top, attenuating the geometry.
            
            final_color = geo_color * vol_transmittance + accumulated_volume_color
            
            color_buffer[x, y] = final_color
            
    return color_buffer

def compute_gradients(volume):
    gradients = empty_array_like(volume)
    for each voxel in volume:
        # Compute differences of values and coordinates
        # X component
        num_x = det([Af0, Ay0, Az0], [Af1, Ay1, Az1], [Af2, Ay2, Az2])
        den_x = det([Ax0, Ay0, Az0], [Ax1, Ay1, Az1], [Ax2, Ay2, Az2])
        gradients.x = num_x / den_x
        
        # Y component (substitute value diffs into 2nd col)
        num_y = det([Ax0, Af0, Az0], [Ax1, Af1, Az1], [Ax2, Af2, Az2])
        den_y = det([Ax0, Ay0, Az0], [Ax1, Ay1, Az1], [Ax2, Ay2, Az2])
        gradients.y = num_y / den_y
        
        # Z component (substitute value diffs into 3rd col)
        num_z = det([Ax0, Ay0, Af0], [Ax1, Ay1, Af1], [Ax2, Ay2, Af2])
        den_z = det([Ax0, Ay0, Az0], [Ax1, Ay1, Az1], [Ax2, Ay2, Az2])
        gradients.z = num_z / den_z
    return gradients
```

## Subtopics

### Hybrid Scanline A-Buffer Volume Rendering
A hybrid rendering system that uses a fast scanline a-buffer algorithm for surface-defined objects and volume rendering for procedural gases, combining them via Z-sorted a-buffer fragments [S2, unofficial]. It creates an a-buffer for a scanline containing a list of fragments for each pixel [S2, unofficial]. Volume rendering is performed for volumes covering the scanline, creating a-buffer fragments for separate sections [S2, unofficial]. Rendering ceases once full coverage is achieved [S2, unofficial]. Fragments are sorted by average Z-depth and rendered to produce the final color [S2, unofficial]. It uses a physically-based low-albedo illumination and atmospheric attenuation model for gases, with volumetric shadows combined via three-dimensional shadow tables [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of volume samples and geometric primitives. Performance is heavily influenced by the complexity of the unstructured grid if used.
*   **Memory Complexity:** High for cell-organized (unstructured) data. A model with 381,548 tetrahedral cells required over 50 Megabytes [S1].
*   **Measured Performance:** On a Stardent ST1000 (4 processors, 5 Mips each, no vector unit used):
    *   Bearing Housing (16,896 tetrahedral cells): 149.2 seconds for a 512x512 image [S1].
    *   CFD Analysis (381,548 tetrahedral cells): 591.6 seconds for a 512x512 image [S1].
*   **Strengths:** Provides clear structural context (e.g., FEM grid) alongside internal density; simple lighting model enhances shape perception [S1].
*   **Limitations:** Wireframe geometry can completely occlude volume data behind it, potentially hiding information [S1]. High memory consumption for unstructured data [S1].

## Background
The method was described in the context of visualizing volumetric data, specifically Finite Element models and Computational Fluid Dynamics (CFD) results [S1]. It was presented as a technique to combine direct volume rendering with standard Z-buffer geometry to show structures like grids or objects about which fluid is flowing [S1]. The work was performed on a Stardent ST1000 and published in 1990 [S1].

## Sources
[S1] Raytracing irregular volume data, https://doi.org/10.1145/99307.99316, fetched 2026-10-25
[S2] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
