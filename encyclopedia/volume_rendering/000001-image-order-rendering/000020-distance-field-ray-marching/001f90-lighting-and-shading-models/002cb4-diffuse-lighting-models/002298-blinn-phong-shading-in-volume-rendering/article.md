# Blinn-Phong Shading in Volume Rendering

Also called: Shaded Volume Rendering (SVR)

## Overview
Blinn-Phong Shading in Volume Rendering is a technique that applies the Blinn-Phong reflectance model to surface normals derived from a Signed Distance Function (SDF) during distance-field ray marching [S1, unofficial]. It distinguishes itself from sibling methods like [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) by using the half-vector approximation for specular highlights, which is computationally cheaper than calculating the reflection vector, and from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular and ambient components to simulate glossy surfaces [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture (`volumeTexture`) containing intensity data, a 1D transfer function texture (`tfTexture`), a uniform transform matrix, light position, light color, and shading parameters (diffuse, specular, ambient, shininess).
*   **Output:** A 2D screen image with accumulated color and alpha.

**Data Structures:**
*   **Uniforms:** A struct containing:
    *   `transform`: `mat4x4<f32>` for transforming screen coordinates to volume space.
    *   `lightPos`: `vec3<f32>` in volume space.
    *   `lightColour`: `vec3<f32>` (RGB).
    *   `diffuse`: `f32` multiplier.
    *   `specular`: `f32` multiplier.
    *   `ambient`: `f32` multiplier.
    *   `tfSize`: `f32` size of the transfer function texture.
*   **Textures:**
    *   `volumeTexture`: `texture_3d<f32>` at binding 0.
    *   `volumeSampler`: Sampler at binding 1.
    *   `tfTexture`: `texture_1d<f32>` at binding 3.

**Steps:**
1.  **Ray Generation:** Generate a full-screen triangle using vertex indices to produce positions $(-1, 3)$, $(-1, -1)$, and $(3, -1)$ in clip space [S1, unofficial]. For each fragment, transform the screen coordinate $(coord.xy, 0.0, 1.0)$ by the uniform transform matrix to get the ray origin $\mathbf{pos}$, and transform $(0.0, 0.0, 1.0, 0.0)$ to get the ray direction $\mathbf{dir}$ [S1, unofficial].
2.  **Ray-Box Intersection:** Compute entry (`near`) and exit (`far`) distances by calculating inverse direction, bounding box distances, and taking min/max of axis intersections. Clamp `near` to 0 if negative [S1, unofficial].
3.  **Marching Loop:** Iterate from `near` to `far`. At each step:
    *   Transform the current position using the uniform matrix and normalize coordinates to the $[0, 1]$ range [S1, unofficial].
    *   Sample the 3D volume texture at these coordinates.
    *   Convert the sampled RG8 format value to a normalized 16-bit intensity: $I = \frac{sample.x + sample.y \times 255}{256}$ [S1, unofficial].
    *   **Normal Calculation:** Estimate the surface normal $\mathbf{n}$ using central differences. For each axis $i \in \{x, y, z\}$, sample the intensity at $\mathbf{p} + \delta_i$ and $\mathbf{p} - \delta_i$, subtract them, and normalize the resulting vector $\mathbf{\Delta}$ [S1, unofficial].
    *   **Transfer Function:** Map the intensity $I$ to a 1D texture index $tfIndex = \lfloor I \times \text{tfSize} \rfloor$ and load the color $\mathbf{c}$ from `tfTexture` [S1, unofficial].
    *   **Lighting:** Calculate light direction $\mathbf{l}$ (from $\mathbf{p}$ to light), view direction $\mathbf{v}$ (from $\mathbf{p}$ to camera), and half vector $\mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|}$ [S1, unofficial]. Compute:
        *   Diffuse: $D = \max(0, \mathbf{n} \cdot \mathbf{l}) \times \text{diffuse}$
        *   Specular: $S = \text{pow}(\max(0, \mathbf{n} \cdot \mathbf{h}), \text{shininess}) \times \text{specular}$
        *   Ambient: $A = \text{ambient}$
        *   Combined: $\mathbf{litColor} = \text{lightColour} \times (\mathbf{c} \times (D + A) + S)$ [S1, unofficial].
    *   **Front-to-Back Composition:** Update accumulated color $\mathbf{accColor}$ using alpha blending:
        *   $\alpha_{new} = (1.0 - \mathbf{accColor}.a) \times \mathbf{c}.a$
        *   $\mathbf{accColor}.rgb += \alpha_{new} \times \mathbf{c}.rgb$
        *   $\mathbf{accColor}.a += \alpha_{new}$
        *   Break if $\mathbf{accColor}.a \ge 0.95$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vert_main(idx: u32) -> vec4<f32>:
    pos_array = [vec2(3.0, -1.0), vec2(-1.0, -1.0), vec2(-1.0, 3.0)]
    return vec4(pos_array[idx], 0.0, 1.0)

# Fragment Shader
def ray_box_intersection(bbox_min: vec3, bbox_max: vec3, ray_pos: vec3, ray_dir: vec3) -> vec2:
    inv_dir = 1.0 / ray_dir
    bot = inv_dir * (bbox_min - ray_pos)
    top = inv_dir * (bbox_max - ray_pos)
    rmin = min(top, bot)
    rmax = max(top, bot)
    t_near = max(rmin.x, max(rmin.y, rmin.z))
    t_far = min(rmax.x, min(rmax.y, rmax.z))
    return vec2(t_near, t_far)

def frag_main(coord: vec4) -> vec4:
    # 1. Ray Generation
    pos = uniforms.transform * vec4(coord.xy, 0.0, 1.0)
    dir = uniforms.transform * vec4(0.0, 0.0, 1.0, 0.0)
    
    # 2. Ray-Box Intersection
    intersect = ray_box_intersection(vec3(0.0), vec3(textureDimensions(volumeTexture)), pos.xyz, dir.xyz)
    near = max(0, i32(intersect.x))
    far = i32(intersect.y)
    
    acc_color = vec4(0.0, 0.0, 0.0, 0.0)
    size = vec3(textureDimensions(volumeTexture))
    
    # 3. Marching Loop
    for k in range(near, far + 1):
        transformed = uniforms.transform * vec4(coord.xy, f32(k), 1.0)
        coords = transformed.xyz / size
        
        sample = textureSample(volumeTexture, volumeSampler, coords)
        intensity = (sample.x + sample.y * 255.0) / 256.0
        
        # Normal Calculation (Central Differences)
        # Note: In a real implementation, offsets must be scaled by volume dimensions
        # and derivatives taken in texture space or world space consistently.
        # Here we assume a simplified gradient estimation based on the source description.
        # The source describes sampling at pos +/- offset, but does not specify the offset magnitude.
        # We assume a small epsilon in normalized coordinates for the sake of completeness.
        eps = 1.0 / size
        p_plus_x = coords + vec3(eps.x, 0.0, 0.0)
        p_minus_x = coords - vec3(eps.x, 0.0, 0.0)
        p_plus_y = coords + vec3(0.0, eps.y, 0.0)
        p_minus_y = coords - vec3(0.0, eps.y, 0.0)
        p_plus_z = coords + vec3(0.0, 0.0, eps.z)
        p_minus_z = coords - vec3(0.0, 0.0, eps.z)
        
        grad_x = (textureSample(volumeTexture, volumeSampler, p_plus_x).x + textureSample(volumeTexture, volumeSampler, p_plus_x).y * 255.0) / 256.0 - \
                 (textureSample(volumeTexture, volumeSampler, p_minus_x).x + textureSample(volumeTexture, volumeSampler, p_minus_x).y * 255.0) / 256.0
        grad_y = (textureSample(volumeTexture, volumeSampler, p_plus_y).x + textureSample(volumeTexture, volumeSampler, p_plus_y).y * 255.0) / 256.0 - \
                 (textureSample(volumeTexture, volumeSampler, p_minus_y).x + textureSample(volumeTexture, volumeSampler, p_minus_y).y * 255.0) / 256.0
        grad_z = (textureSample(volumeTexture, volumeSampler, p_plus_z).x + textureSample(volumeTexture, volumeSampler, p_plus_z).y * 255.0) / 256.0 - \
                 (textureSample(volumeTexture, volumeSampler, p_minus_z).x + textureSample(volumeTexture, volumeSampler, p_minus_z).y * 255.0) / 256.0
                 
        normal = normalize(vec3(grad_x, grad_y, grad_z))
        
        # Transfer Function
        tf_index = i32(intensity * uniforms.tfSize)
        color = textureSample(tfTexture, volumeSampler, vec3(f32(tf_index) / uniforms.tfSize, 0.0, 0.0))
        
        # Lighting
        light_dir = normalize(uniforms.lightPos - pos.xyz)
        view_dir = normalize(-pos.xyz) # Assuming camera at origin in volume space for simplicity, or derived from transform
        half_dir = normalize(light_dir + view_dir)
        
        diffuse = max(0.0, dot(normal, light_dir)) * uniforms.diffuse
        specular = pow(max(0.0, dot(normal, half_dir)), 10.0) * uniforms.specular # shininess = 10.0
        ambient = uniforms.ambient
        
        lit_color = uniforms.lightColour * (color.rgb * (diffuse + ambient) + specular)
        
        # Front-to-Back Composition
        alpha = (1.0 - acc_color.a) * color.a
        acc_color.rgb += alpha * lit_color.rgb
        acc_color.a += alpha
        
        if acc_color.a >= 0.95:
            break
            
    return acc_color
```

**Parameters:**
*   `shininess`: 10.0 [S1, unofficial].
*   `diffuse`, `specular`, `ambient`: Set via uniforms, typical values not specified in sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The intensity conversion from RG8 to 16-bit float uses integer arithmetic in the source description, which may introduce quantization artifacts if not handled carefully in the shader [S1, unofficial].
*   **Boundaries:** The ray-box intersection clamps `near` to 0, which may cause artifacts if the ray origin is inside the volume but the intersection logic assumes an external origin [S1, unofficial].
*   **Normal Estimation:** Central differences require sampling at offsets. The source does not specify the offset size, which is critical for correct normal estimation in discrete volumes [S1, unofficial].

## Subtopics

### Phong Shading with SDF Normals
Uses the reflection vector instead of the half vector for specular highlights, offering more accurate reflections but higher computational cost. Choose when visual fidelity of specular highlights is critical. [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md)

### Lambertian Lighting in SDF
Only computes diffuse lighting, ignoring specular and ambient components. Choose for matte, non-reflective surfaces or when performance is paramount. [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)

### Gouraud Shading
Computes lighting at vertices and interpolates across the surface, which is less accurate for curved surfaces but faster. Choose for low-poly approximations or very high-performance requirements. [Gouraud Shading](4645-gouraud-shading.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, which depends on the volume depth and step size. The breaking condition at 95% opacity can reduce average samples.
*   **Memory Complexity:** Low, requiring only the volume texture, transfer function texture, and uniform buffers.
*   **Measured Performance:** Implemented in `ossium` for WebGPU, running in the browser. Specific performance metrics (FPS, resolution) are not provided in the sources [S1, unofficial].
*   **Strengths:** Real-time capable on modern GPUs; provides visually appealing glossy surfaces; simple to implement.
*   **Limitations:** Approximate specular highlights (Blinn-Phong); normal estimation can be noisy in discrete volumes; transfer function mapping is linear and may not capture complex material properties.
*   **When to Use:** Real-time volume visualization where glossy appearance is desired and performance is a constraint.
*   **When Not to Use:** Scientific visualization requiring physically accurate light transport or high-fidelity normal estimation.

## Background
Blinn-Phong Shading in Volume Rendering was implemented in the `ossium` project, a WebGPU-based application for displaying 3D volumes created from DICOM files in the browser [S1, unofficial]. The technique is part of the broader field of Shaded Volume Rendering (SVR), which enhances traditional volume rendering by adding surface shading cues to improve depth perception and material appearance.

## Sources
[S1] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
