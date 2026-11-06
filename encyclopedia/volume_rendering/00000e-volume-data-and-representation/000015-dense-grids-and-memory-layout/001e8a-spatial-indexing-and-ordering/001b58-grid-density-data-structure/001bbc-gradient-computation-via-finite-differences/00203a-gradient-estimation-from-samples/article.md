# Gradient Estimation from Samples

Also called: Gradient-based Normal Estimation, Gradient-based Volume Shading, Multi-Volume Ray Casting Gradient Estimation and Shading, Normal Computation for Shading, Normal Vector Computation, Surface Normal Estimation, vtkGPUMultiVolumeRayCastMapper shading logic

## Overview
Gradient Estimation from Samples is a technique for computing surface normals directly from the scalar field samples taken during volume ray casting, enabling Phong-based shading without precomputing a separate gradient volume [S1, unofficial][S3]. It differs from [Precomputed Gradient Volume](2425-precomputed-gradient-volume.md) by avoiding the memory overhead and preprocessing time of storing explicit gradient vectors, and from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by relying on discrete finite differences rather than closed-form derivatives [S3]. This method is primarily used in interactive GPU-based volume rendering to enhance shape perception through diffuse and specular lighting cues [S1, unofficial][S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D texture (`dataSetTexture`) containing the scalar volume data, sampled at a current ray position `posX`.
*   **Output:** A shaded color value (`finalColor`) for the current sample, incorporating lighting and shading based on the estimated normal.

**Data Structures:**
*   `dataSetTexture`: An array of 3D samplers (`sampler3D`) used for volume data sampling [S2, unofficial].
*   `colorTexture`: An array of 1D samplers (`uniform sampler1D colorTexture[MAX_NUMBER_OF_VOLUMES]`) used for color mapping in one-component datasets. Max size is 4 on Apple platforms and 5 otherwise [S2, unofficial].
*   `transposeTextureToEye`: A matrix used to transform the normalized gradient from texture space to eye space [S2, unofficial].
*   `cellScale`: A scalar used to scale the gradient vector from texture coordinates to eye coordinates [S2, unofficial].

**Algorithm Steps:**
1.  **Extract Scalar Value:** Determine the scalar value $f$ at the current position. For a 4-component dataset, use the W component; for 1-component, use the X component [S2, unofficial].
    $$ \text{scalar} = \begin{cases} \text{value}.w & \text{if 4 components} \\ \text{value}.x & \text{if 1 component} \end{cases} $$
2.  **Compute Gradient Components:** Calculate the central difference gradient vector $\mathbf{g}$ by sampling the texture at offsets from the current position `posX`. Let $\mathbf{xvec}, \mathbf{yvec}, \mathbf{zvec}$ be the unit vectors in texture space corresponding to the x, y, and z axes [S2, unofficial].
    $$ \mathbf{g}_{\text{raw}} = \begin{bmatrix} f(\text{posX} + \mathbf{xvec}) - f(\text{posX} - \mathbf{xvec}) \\ f(\text{posX} + \mathbf{yvec}) - f(\text{posX} - \mathbf{yvec}) \\ f(\text{posX} + \mathbf{zvec}) - f(\text{posX} - \mathbf{zvec}) \end{bmatrix} $$
    *Note: The source code samples `posX + vec` and `posX - vec` separately into `g1` and `g2` before subtraction logic is implied by the central difference formula [S1, S2, unofficial].*
3.  **Scale Gradient:** Convert the gradient from texture space to eye coordinates by scaling with `cellScale` [S2, unofficial].
    $$ \mathbf{g}_{\text{eye}} = \mathbf{g}_{\text{raw}} \times \text{cellScale} $$
4.  **Normalize Gradient:** Compute the length $L = \|\mathbf{g}_{\text{eye}}\|$. If $L > 0$, transform the gradient to eye space using the inverse texture-to-eye matrix `transposeTextureToEye` and normalize:
    $$ \mathbf{n} = \frac{\text{transposeTextureToEye} \times \mathbf{g}_{\text{eye}}}{L} $$
    If $L = 0$, set $\mathbf{n} = (0, 0, 0)$ [S2, unofficial].
5.  **Compute Lighting Vectors:**
    *   If the light is positional ($w \neq 0$):
        *   Transform vertex position `hPos` to texture coordinates: $\mathbf{tmp} = \text{eyeToTexture4} \times \text{hPos}$.
        *   Convert to eye coordinates: $\mathbf{ldir}_{\text{dir}} = \mathbf{tmp}.xyz / \mathbf{tmp}.w$.
        *   Calculate light direction: $\mathbf{ldir} = \text{lightPos} - \mathbf{ldir}_{\text{dir}}$.
        *   Calculate squared distance: $d^2 = \mathbf{ldir} \cdot \mathbf{ldir}$.
        *   Normalize: $\mathbf{ldir} = \mathbf{ldir} / \sqrt{d^2}$.
        *   Calculate half vector: $\mathbf{h} = \text{normalize}(\mathbf{ldir} + \mathbf{wReverseRayDir})$ [S2, unofficial].
    *   If the light is directional ($w = 0$), $\mathbf{ldir}$ is taken directly from the light source direction [S2, unofficial].
6.  **Calculate Attenuation:**
    $$ \text{att} = \frac{1.0}{C + L \sqrt{d^2} + Q d^2} $$
    Where $C, L, Q$ are constant, linear, and quadratic attenuation factors. If the light is directional, $\text{att} = 1.0$ [S2, unofficial].
7.  **Calculate Spot Illumination:**
    *   If spot cutoff is $180^\circ$, $\text{spot} = 1.0$.
    *   Otherwise, calculate $\text{coef} = -\mathbf{ldir} \cdot \text{spotDirection}$.
    *   If $\text{coef} \ge \text{spotCosCutoff}$, $\text{spot} = \text{coef}^{\text{spotExponent}}$.
    *   Else, $\text{spot} = 0.0$ [S2, unofficial].
8.  **Shade:**
    *   Initialize `finalColor` to $(0, 0, 0, 0)$ [S2, unofficial].
    *   Calculate dot products: $nDotL = \mathbf{n} \cdot \mathbf{ldir}$ and $nDotH = \mathbf{n} \cdot \mathbf{h}$ [S2, unofficial].
    *   **Two-sided shading:** If $nDotL < 0$, negate $nDotL$. If $nDotH < 0$, negate $nDotH$ [S2, unofficial].
    *   Add ambient: $\text{finalColor} += \text{ambientProduct}$ [S2, unofficial].
    *   Add diffuse (if $nDotL > 0$): $\text{finalColor} += (\text{diffuseProduct} \times nDotL) \times \text{color}$ [S2, unofficial].
    *   Add specular: $\text{shininessFactor} = nDotH^{\text{shininess}}$. $\text{finalColor} += \text{specularProduct} \times \text{shininessFactor}$ [S2, unofficial].
    *   Apply attenuation and spot: $\text{finalColor} *= \text{att} \times \text{spot}$ [S2, unofficial].
    *   Add scene ambient: $\text{finalColor} += \text{sceneColor} \times \text{color}$ [S2, unofficial].
9.  **Clamp:** Clamp `finalColor` to `[clampMin, clampMax]` to prevent black spots [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shade_volume_sample(pos, ray_dir, light_pos, light_dir, light_constants, 
                        transpose_texture_to_eye, cell_scale, clamp_min, clamp_max,
                        shininess, color_value, is_directional_light):
    """
    Computes shaded color for a volume sample using gradient estimation from samples.
    """
    # 1. Extract scalar value
    # Assuming 4-component dataset uses W, 1-component uses X
    scalar = color_value.w if color_value.w is not None else color_value.x
    
    # 2. Compute Gradient (Central Differences)
    # Offsets are unit vectors in texture space
    x_vec = vec3(1.0, 0.0, 0.0)
    y_vec = vec3(0.0, 1.0, 0.0)
    z_vec = vec3(0.0, 0.0, 1.0)
    
    # Sample neighbors
    val_pos_x = texture3D(dataSetTexture, pos + x_vec)
    val_neg_x = texture3D(dataSetTexture, pos - x_vec)
    val_pos_y = texture3D(dataSetTexture, pos + y_vec)
    val_neg_y = texture3D(dataSetTexture, pos - y_vec)
    val_pos_z = texture3D(dataSetTexture, pos + z_vec)
    val_neg_z = texture3D(dataSetTexture, pos - z_vec)
    
    # Extract scalar components (assuming same component logic as input)
    s_pos_x = val_pos_x.w if val_pos_x.w is not None else val_pos_x.x
    s_neg_x = val_neg_x.w if val_neg_x.w is not None else val_neg_x.x
    s_pos_y = val_pos_y.w if val_pos_y.w is not None else val_pos_y.x
    s_neg_y = val_neg_y.w if val_neg_y.w is not None else val_neg_y.x
    s_pos_z = val_pos_z.w if val_pos_z.w is not None else val_pos_z.x
    s_neg_z = val_neg_z.w if val_neg_z.w is not None else val_neg_z.x
    
    # Central difference formula
    grad_raw = vec3(
        (s_pos_x - s_neg_x),
        (s_pos_y - s_neg_y),
        (s_pos_z - s_neg_z)
    )
    
    # 3. Scale Gradient
    grad_eye = grad_raw * cell_scale
    
    # 4. Normalize Gradient
    normal_length = length(grad_eye)
    if normal_length > 0.0:
        # Transform to eye space and normalize
        grad_eye = transpose_texture_to_eye * grad_eye
        normal = normalize(grad_eye)
    else:
        normal = vec3(0.0, 0.0, 0.0)
        
    # 5. Compute Lighting Vectors
    if not is_directional_light:
        # Positional light
        # Transform position to texture coords then to eye coords for light direction calc
        # Note: eyeToTexture4 is assumed available from context
        tmp = eye_to_texture4 * pos
        ldir_dir = tmp.xyz / tmp.w
        ldir = light_pos - ldir_dir
        sqr_distance = dot(ldir, ldir)
        ldir = normalize(ldir)
        # Half vector calculation requires reverse ray direction
        h = normalize(ldir + ray_dir) # Assuming ray_dir is the reverse ray direction or wReverseRayDir
    else:
        ldir = light_dir
        sqr_distance = 0.0 # Infinite distance for directional light
        h = normalize(ldir + ray_dir)
        
    # 6. Calculate Attenuation
    if not is_directional_light:
        att = 1.0 / (light_constants.constant + light_constants.linear * sqrt(sqr_distance) + light_constants.quadratic * sqr_distance)
    else:
        att = 1.0
        
    # 7. Calculate Spot Illumination
    if light_constants.spot_cutoff == 180.0:
        spot = 1.0
    else:
        coef = -dot(ldir, light_constants.spot_direction)
        if coef >= light_constants.spot_cos_cutoff:
            spot = pow(coef, light_constants.spot_exponent)
        else:
            spot = 0.0
            
    # 8. Shade
    final_color = vec4(0.0, 0.0, 0.0, 0.0)
    
    n_dot_l = dot(normal, ldir)
    n_dot_h = dot(normal, h)
    
    # Two-sided shading
    if n_dot_l < 0.0:
        n_dot_l = -n_dot_l
    if n_dot_h < 0.0:
        n_dot_h = -n_dot_h
        
    # Ambient
    final_color += light_constants.ambient_product
    
    # Diffuse
    if n_dot_l > 0.0:
        final_color += (light_constants.diffuse_product * n_dot_l) * color_value
        
    # Specular
    shininess_factor = pow(n_dot_h, shininess)
    final_color += light_constants.specular_product * shininess_factor
    
    # Apply attenuation and spot
    final_color *= att * spot
    
    # Scene Ambient
    final_color += light_constants.scene_color * color_value
    
    # 9. Clamp
    final_color = clamp(final_color, clamp_min, clamp_max)
    
    return final_color
```

**Parameters:**
*   `clampMin`, `clampMax`: Lower and upper bounds for clamping the final color to avoid black spots [S2, unofficial].
*   `cellScale`: Scalar to convert gradient from texture to eye coordinates [S2, unofficial].
*   `transposeTextureToEye`: Matrix for transforming normalized gradient to eye space [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Black Spots:** Occur if the final color is not clamped or if negative dot products are not handled for two-sided shading [S2, unofficial].
*   **Zero Gradient:** If the gradient length is zero, the normal is set to $(0,0,0)$, resulting in no diffuse/specular contribution [S2, unofficial].
*   **Boundary Conditions:** Central differences require neighbors. At volume boundaries, sampling outside the texture domain must be handled (e.g., clamping or repeating), though the source code assumes valid texture sampling [S2, unofficial].

## Subtopics

### [Gradient-based Pre-computed Shading](3159-gradient-based-pre-computed-shading.md)
Precomputes shaded colors for each voxel using a Lambertian model to avoid per-ray shading costs. Choose when rendering performance is critical and view independence is acceptable.

### [Gradient-Based Surface Shading](5648-gradient-based-surface-shading.md)
Approximates surface normals at query points for enhanced shape perception. Choose when intrinsic spatial variation cues are needed without external light sources.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it requires a constant number of texture lookups (6 for central differences) and vector operations.
*   **Memory Complexity:** $O(1)$ additional memory per sample, as no precomputed gradient volume is stored [S3].
*   **Performance:** Enables interactive visualization on GPU workstations by avoiding expensive precomputation [S3]. However, it is computationally more expensive than simple dot-product shading due to the gradient evaluation [S1, unofficial].
*   **Strengths:** No preprocessing time, low memory overhead, enhances shape perception [S1, unofficial][S3].
*   **Limitations:** Gradient estimation can be noisy in sparse or low-resolution regions; central differences may be inaccurate near boundaries [S1, unofficial][S3].

## Background
The method was introduced in the context of interactive volume rendering for medical data visualization, using central differences to estimate normals for Phong shading [S1, unofficial]. It was further implemented in the VTK `vtkGPUMultiVolumeRayCastMapper` for multi-volume ray casting, allowing for real-time shading on GPUs [S2, unofficial]. A related contribution by Wald et al. described a novel method to compute gradient vectors from just the samples taken for reconstruction in AMR data, highlighting its utility for interactive visualization [S3].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S3] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
