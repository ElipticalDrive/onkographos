# Simplified Phong Lighting Model

Also called: Phong Illumination, Phong Lighting Model, Phong Reflection Model, Phong Illumination Model

## Overview
The Simplified Phong Lighting Model is a local illumination technique adapted for volume rendering to enhance the perception of 3D structure by simulating surface-like shading on scalar field data [S2, unofficial]. It solves the problem of flat, unlit volume visualization by calculating diffuse and specular contributions based on the gradient of the volume data, which serves as an approximation of the surface normal [S2, S3, S4, S5, S7, unofficial]. Unlike physically based light transport methods, it is a heuristic shading model that modulates the existing pixel color or volume samples to create highlights and shadows, distinguishing itself from its siblings by typically relying on a single light source and explicit vector math rather than precomputed spectral expansions or complex phase functions [S3, S4, S5, S7, unofficial].

## How it works
The method operates by iterating along rays cast through the volume data [S2, unofficial]. For each ray, it identifies the first significant voxel where the volume value exceeds a threshold, computes the local surface normal from the gradient of the scalar field, and applies the Phong reflection model to calculate the final color contribution [S2, S3, S4, S5, S7, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `volume`: A 3D scalar field (voxel grid) containing density or intensity values [S2, S3, S4, S5, unofficial].
    *   `view_matrix`: A 4x4 matrix defining the camera orientation and projection, used to derive view vectors [S2, S4, S5, unofficial].
    *   `light_position`: The 3D position of the light source in world or texture space [S3, S4, S5, unofficial].
    *   `light_intensity`: The color and intensity of the light source [S3, S5, unofficial].
    *   `ambient_light`: The global ambient light color/intensity [S3, S5, unofficial].
    *   `material_properties`: Reflectance coefficients for ambient ($k_a$), diffuse ($k_d$), and specular ($k_s$) lighting, and the shininess exponent ($n$ or $\alpha$) [S3, S5, S7, unofficial].
*   **Outputs**:
    *   `color`: The final RGB color value for the pixel, combining ambient, diffuse, and specular components [S3, S5, S7, unofficial].

### Data Structures
*   **Voxel**: A single element in the volume grid, accessed via integer coordinates $(x, y, z)$ [S2, unofficial].
*   **Vector**: 3D floating-point vectors for normals ($N$), light direction ($L$), view direction ($V$ or $r$), and reflection direction ($R$ or $v$) [S3, S4, S5, unofficial].
*   **Material**: A structure or set of parameters holding $k_a, k_d, k_s$, and shininess [S3, S5, unofficial].

### Numbered Steps

1.  **Initialization**:
    *   Initialize the output color to the ambient component: $C = k_a \cdot I_a$ [S3, S5, S7, unofficial].
    *   Initialize a flag `surface_found` to `False` [S2, unofficial].
    *   Initialize the normal vector $N$ to zero [S2, unofficial].

2.  **Ray Casting and Surface Detection**:
    *   Iterate along the ray from the camera plane into the volume [S2, unofficial]. The iteration step size $k$ can be fixed (e.g., 50) for performance or 1 for accuracy [S2, unofficial].
    *   For each step, calculate the voxel coordinates $(x, y, z)$ using the view vector and pixel base position [S2, S4, unofficial].
    *   Sample the voxel value $v$ at $(x, y, z)$ [S2, S3, unofficial].
    *   If $v > 1$ (or a defined threshold) and `surface_found` is `False`:
        *   Set `surface_found` to `True` [S2, unofficial].
        *   Compute the gradient $\nabla f = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ at the current coordinates [S2, S5, unofficial].
        *   Calculate the magnitude $|\nabla f|$ [S2, unofficial].
        *   Normalize the gradient to obtain the surface normal: $N = \frac{\nabla f}{|\nabla f|}$ [S2, S4, S5, unofficial].
        *   Break the loop or continue for compositing, depending on the specific implementation (e.g., ray marching vs. first-hit) [S2, S3, unofficial].

3.  **Vector Calculation**:
    *   Calculate the normalized light direction vector $L$:
        $$L = \text{normalize}(\text{light\_position} - \text{point})$$ [S3, S4, S5, unofficial].
    *   Calculate the normalized view direction vector $V$ (from point to eye):
        $$V = \text{normalize}(\text{eye\_position} - \text{point})$$ [S3, S4, S5, unofficial].
        *Note: Some implementations use the variable name $r$ for the view vector and $v$ for the reflection vector, which is non-standard but functionally equivalent if consistent [S5, unofficial].*

4.  **Diffuse Calculation**:
    *   Compute the dot product between the normal and light direction:
        $$dot_{LN} = N \cdot L$$ [S3, S4, S5, S7, unofficial].
    *   Apply the facing parameter: If $dot_{LN} < 0$, the surface is facing away from the light [S5, unofficial].
    *   Otherwise, calculate the diffuse factor:
        $$diffuse\_factor = \max(dot_{LN}, 0.0)$$ [S3, S4, S5, unofficial].
    *   Calculate the diffuse color contribution:
        $$C_{diffuse} = k_d \cdot I_l \cdot diffuse\_factor$$ [S3, S5, S7, unofficial].

5.  **Specular Calculation**:
    *   Calculate the reflection vector $R$ of the light direction around the normal:
        $$R = \text{reflect}(-L, N)$$ [S3, S4, S5, unofficial].
    *   Compute the dot product between the view direction and the reflection vector:
        $$dot_{RV} = V \cdot R$$ [S3, S4, S5, unofficial].
    *   Apply the facing check for specular: If $dot_{RV} < 0$, the reflection is not visible to the viewer [S5, unofficial].
    *   Otherwise, calculate the specular factor using the shininess exponent $n$ (or $\alpha$):
        $$specular\_factor = \max(dot_{RV}, 0.0)^n$$ [S3, S4, S5, unofficial].
    *   Calculate the specular color contribution:
        $$C_{specular} = k_s \cdot I_l \cdot specular\_factor$$ [S3, S5, S7, unofficial].

6.  **Final Color Composition**:
    *   Sum the components:
        $$C_{final} = C_{ambient} + C_{diffuse} + C_{specular}$$ [S3, S5, S7, unofficial].
    *   If using a simplified shadowing approach based on the angle between normal and view (as in [S2, unofficial]), multiply the existing pixel color by a shadow factor derived from $N \cdot V$ mapped to $[0, 1]$. However, standard Phong models do not include self-shadowing [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simplified_phong_lighting(volume, view_matrix, light_position, light_intensity, ambient_light, material, eye_position):
    """
    Computes the color for a single ray/sample point using the Simplified Phong Lighting Model.
    """
    # 1. Ambient Component
    color = material['ka'] * ambient_light
    
    # 2. Find Surface and Compute Normal
    surface_found = False
    normal = None
    point = None
    
    image_size = view_matrix.shape[0]
    image_center = image_size / 2.0
    u_vector = view_matrix[0:3]
    v_vector = view_matrix[4:7]
    volume_center = [volume.dim_x / 2, volume.dim_y / 2, volume.dim_z / 2]
    
    # Ray casting loop with step size 50 for performance [S2, unofficial]
    for k in range(0, image_size, 50):
        voxel_coordinate_x = u_vector[0] * (k - image_center) + v_vector[0] * (k - image_center) + volume_center[0]
        voxel_coordinate_y = u_vector[1] * (k - image_center) + v_vector[1] * (k - image_center) + volume_center[1]
        voxel_coordinate_z = u_vector[2] * (k - image_center) + v_vector[2] * (k - image_center) + volume_center[2]
        
        value = volume[voxel_coordinate_x, voxel_coordinate_y, voxel_coordinate_z]
        
        if value > 1.0 and not surface_found:
            surface_found = True
            point = (voxel_coordinate_x, voxel_coordinate_y, voxel_coordinate_z)
            delta_f = compute_gradient(volume, point)
            magnitude_f = math.sqrt(delta_f[0]**2 + delta_f[1]**2 + delta_f[2]**2)
            normal = (delta_f[0] / magnitude_f, delta_f[1] / magnitude_f, delta_f[2] / magnitude_f)
            break
            
    if not surface_found or normal is None:
        return color

    # 3. Vector Calculations
    L = normalize(light_position - point)
    V = normalize(eye_position - point)
    
    # 4. Diffuse Component
    dot_LN = dot(normal, L)
    
    if dot_LN < 0.0:
        diffuse = vec3(0.0, 0.0, 0.0)
    else:
        diffuse_factor = dot_LN
        diffuse = material['kd'] * light_intensity * diffuse_factor
        
    # 5. Specular Component
    R = reflect(-L, normal)
    dot_RV = dot(V, R)
    
    if dot_RV < 0.0:
        specular = vec3(0.0, 0.0, 0.0)
    else:
        specular_factor = pow(max(dot_RV, 0.0), material['shininess'])
        specular = material['ks'] * light_intensity * specular_factor
        
    # 6. Final Color
    color += diffuse + specular
    
    return color
```

### Parameters and Typical Values
*   **Shininess ($n$ or $\alpha$)**: Controls the size of the specular highlight. Higher values produce smaller, sharper highlights [G]. Typical values range from 1 to 1000, depending on the desired material appearance [S3, S5, S7, unofficial].
*   **Light Position**: Often set to a fixed world coordinate, e.g., $(8, 20, 8)$ or $(0, 20, 0)$ [S3, S5, unofficial].
*   **Ambient Light**: Often a dim white or gray, e.g., $0.5 \cdot (1,1,1)$ or $0.025 \cdot (0.8, 0.8, 0.8)$ [S3, S5, unofficial].
*   **Light Intensity**: Often set to $(0.8, 0.8, 0.8)$ or $(0.2, 0.4, 0.4)$ [S3, S5, unofficial].
*   **Step Size**: In interactive modes, the ray marching step can be increased (e.g., 50) for performance, while production renders use 1 [S2, unofficial].

### Edge Cases and Pitfalls
*   **Self-Shadowing**: Volume self-shadowing is computationally expensive and is typically ignored in this simplified model to maintain performance and clarity [S1, unofficial].
*   **Gradient Noise**: The normal vector derived from the gradient can be noisy in low-resolution or noisy volume data, leading to flickering specular highlights [S2, S5, unofficial].
*   **Facing Direction**: The model correctly handles back-facing surfaces by returning zero diffuse/specular contribution when $N \cdot L < 0$ [S3, S5, S7, unofficial].
*   **Reflection Visibility**: Specular highlights are only visible if the reflection vector points towards the viewer ($V \cdot R > 0$) [S3, S5, unofficial].

## Subtopics

### [Specular Reflections via Polynomial Expansion](1172-specular-reflections-via-polynomial-expansion.md)
Extends the Fourier Projection-Slice technique to support Phong specular reflections by expanding the specular term into a polynomial, allowing evaluation via precomputed Fourier spectra without per-voxel shading.

### [Phong Shading Model for Volume Data](1353-phong-shading-model-for-volume-data.md)
Applies the Phong model to volume data projected onto 2D/3D textures, leveraging hardware-accelerated texture lookups and blending for interactive performance on older hardware.

### [Phong Shading](1853-phong-shading.md)
The standard local illumination model applied to volume samples, using the gradient as the normal vector and explicit vector math for reflection and view directions.

### [Blinn-Phong Shading Model](2239-blinn-phong-shading-model.md)
Uses the half-vector between light and view directions for specular calculation, avoiding the expensive reflection vector calculation and providing smoother lobes.

### [Blinn-Phong Lighting with Depth Cueing](2448-blinn-phong-lighting-with-depth-cueing.md)
Combines Blinn-Phong shading with depth cueing to enhance depth perception and simulate atmospheric perspective by fading distant voxels.

### Final Illumination and Texture Mapping
This variant integrates the Simplified Phong Lighting Model with texture mapping and environment mapping [S8, unofficial]. It utilizes built-in trilinear texture interpolation to smoothly reconstruct normal vectors from attribute maps, ensuring continuous per-fragment normals [S8, unofficial]. These normals are then used for Phong shading, as well as for implementing reflection and refraction via cube mapping [S8, unofficial]. The final fragment color is computed by summing contributions from lighting, texture mapping, reflectance mapping, and refraction mapping in a single rendering pass, avoiding multiple render targets or additional blending passes [S8, unofficial].

## Operating characteristics
*   **Performance**: Fast and suitable for real-time or interactive visualization, especially when using larger step sizes or hardware-accelerated texture lookups [S2, S3, unofficial].
*   **Strengths**: Simple to implement, computationally efficient, enhances visual perception of 3D structure, and works well with scalar field data [S2, S3, S4, S5, S7, unofficial].
*   **Limitations**: Occlusion shadows are typically ignored to maintain performance and clarity [S1, unofficial]. Gradient-derived normals can be noisy in low-resolution data [S2, S5, unofficial].
*   **When to Use**: Interactive visualization and non-photorealistic rendering where conveying information is prioritized over photographic realism [S1, S2, S3, unofficial].
*   **When Not to Use**: Scientific visualization requiring quantitative accuracy, or scenes with complex lighting interactions like shadows and reflections [S1, unofficial].

## Background
The Phong Lighting Model was originally developed by Bui Tuong Phong for polygonal surfaces [S2, S4, S5, S7, unofficial]. Its application to volume rendering emerged as a heuristic to enhance the visual quality of scalar field data, which otherwise appears flat and unstructured [S2, S3, S4, S5, S7, unofficial]. The simplified version for volumes typically involves computing the gradient of the scalar field to approximate surface normals and applying the standard Phong reflection model to these normals [S2, S3, S4, S5, S7, unofficial]. This approach has been widely adopted in non-photorealistic rendering (NPR) and interactive volume visualization systems due to its simplicity and effectiveness [S1, S2, S3, S4, S5, S7, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S3] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S4] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S5] Adam-Gleave/sdf-terrain, https://github.com/Adam-Gleave/sdf-terrain, fetched 2026-10-26 (unofficial)
[S6] HeartofTheForce/Isosurface, https://github.com/HeartofTheForce/Isosurface, fetched 2026-10-26 (unofficial)
[S7] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S8] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
