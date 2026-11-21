# Phong Shading with Volume Rendering
Also called: compositeShaderPhong

## Overview
Phong Shading with Volume Rendering enhances standard direct volume ray casting by applying Phong lighting models (diffuse and specular) to volume samples based on computed surface normals derived from volume gradients [S1, unofficial]. Unlike standard compositing which treats samples as opaque or translucent colored blobs, this method calculates surface-like shading interactions before compositing, adding geometric depth cues to the volume [S1, unofficial]. It differs from [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md) by focusing on local surface reflection rather than volumetric light transport, and from [Intermixing](5139-intermixing.md) by using explicit gradient-based normals rather than structural analysis.

## How it works
**Inputs:**
- Volume dataset (scalar field).
- Transfer function mapping scalar values to RGBA (color $c$, opacity $\alpha$).
- Camera position ($v_{position}$).
- Light direction vector ($lightVector$).
- Phong parameters: shininess exponent, specular coefficient ($k_s$).
- Ray origin and direction.

**Outputs:**
- Final pixel color $C_{final}$.

**Data Structures:**
- Sample point: 3D position $(x, y, z)$.
- Sample color: RGB vector $c$.
- Sample opacity: float $\alpha$.
- Gradient: 3D vector $\nabla V$ (computed via finite differences or trilinear derivative of the transfer function input).

**Steps:**
1. **Ray-Volume Intersection:** Perform standard Axis-Aligned Bounding Box (AABB) intersection to determine the entry parameter $t_0$ and exit parameter $t_1$ [S1, unofficial]. Calculate the step size (sample length) based on the desired number of samples or voxel resolution.
2. **Initialization:** Initialize the color accumulator $C_{acc} = (0, 0, 0)$ and the accumulated opacity $\alpha_{acc} = 0.0$ [S1, unofficial]. Initialize transmittance $T = 1.0$.
3. **Ray Casting Loop:** Iterate along the ray from $t_0$ to $t_1$ for a fixed number of steps or until $\alpha_{acc} \ge 0.99$ [S1, unofficial].
    a. **Sample Retrieval:** At the current ray position, retrieve the scalar value from the volume. Use the transfer function to obtain the sample color $c_i$ (RGB) and opacity $\alpha_i$.
    b. **Normal Computation:** Compute the gradient vector $\mathbf{n}$ at the current position. This is typically done by sampling the scalar field at neighboring points and calculating the finite difference, or by differentiating the transfer function's input mapping. If the gradient magnitude is zero, skip lighting calculations for this sample.
    c. **Lighting Calculation:**
        i. Normalize the gradient to get the surface normal $\mathbf{N} = \frac{\mathbf{n}}{\|\mathbf{n}\|}$.
        ii. Compute the view direction vector $\mathbf{V}$ from the sample point to the camera position ($v_{position}$), and normalize it.
        iii. Normalize the light direction vector $\mathbf{L} = lightVector$.
        iv. Compute the diffuse term: $D = \max(0, \mathbf{N} \cdot \mathbf{L})$.
        v. Compute the reflection vector $\mathbf{R} = 2(\mathbf{N} \cdot \mathbf{L})\mathbf{N} - \mathbf{L}$.
        vi. Compute the specular term: $S = k_s \cdot (\max(0, \mathbf{R} \cdot \mathbf{V}))^{\text{shininess}}$.
        vii. Modify the sample color: $c'_i = c_i \cdot (0.05 + D) + S$. The term $0.05 \cdot c_i$ acts as an ambient-like base illumination [S1, unofficial].
    d. **Compositing:** Apply the modified color $c'_i$ and original opacity $\alpha_i$ to the front-to-back compositing equation:
       $$ C_{acc} \leftarrow C_{acc} + T \cdot \alpha_i \cdot c'_i $$
       $$ \alpha_{acc} \leftarrow \alpha_{acc} + T \cdot \alpha_i $$
       $$ T \leftarrow T \cdot (1 - \alpha_i) $$
    e. **Termination Check:** If $\alpha_{acc} \ge 0.99$, set $\alpha_{acc} = 1.0$ and break the loop [S1, unofficial].
    f. **Advance:** Move the ray position forward by the sample length.
4. **Background Composite:** Composite the result over the background color $C_{bg}$:
   $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_volume_shading(ray_origin, ray_direction, volume_data, transfer_function, camera_pos, light_dir, shininess, ks):
    """
    Performs volume ray casting with Phong shading.
    """
    # 1. AABB Intersection
    t0, t1 = intersect_aabb(ray_origin, ray_direction, volume_bounds)
    if t1 < t0:
        return background_color
    
    num_samples = calculate_num_samples(t1 - t0, voxel_size)
    step_size = (t1 - t0) / num_samples
    
    # 2. Initialization
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    T = 1.0
    
    # 3. Ray Casting Loop
    for i in range(num_samples):
        t = t0 + i * step_size
        pos = ray_origin + t * ray_direction
        
        # a. Sample Retrieval
        scalar_val = sample_volume(pos, volume_data)
        c_i, alpha_i = transfer_function(scalar_val)
        
        # b. Normal Computation
        grad = compute_gradient(pos, volume_data)
        grad_norm = length(grad)
        
        if grad_norm > 1e-6:
            N = normalize(grad)
            
            # c. Lighting Calculation
            V = normalize(camera_pos - pos)
            L = normalize(light_dir)
            
            # Diffuse
            diff = max(0.0, dot(N, L))
            
            # Specular
            R = reflect(-L, N) # R = 2(N.L)N - L
            spec = ks * pow(max(0.0, dot(R, V)), shininess)
            
            # Modify color
            # Ambient-like term: 0.05 * c_i
            c_i = c_i * (0.05 + diff) + spec
        else:
            # No normal, use unlit color
            pass
            
        # d. Compositing
        C_acc = C_acc + T * alpha_i * c_i
        alpha_acc = alpha_acc + T * alpha_i
        T = T * (1.0 - alpha_i)
        
        # e. Termination
        if alpha_acc >= 0.99:
            alpha_acc = 1.0
            break
            
    # 4. Background Composite
    return C_acc + (1.0 - alpha_acc) * background_color
```

**Parameters:**
- `shininess`: The specular exponent for the Phong model. Typical values range from 10 to 100 depending on desired glossiness [S1, unofficial].
- `ks`: The specular coefficient multiplier. Controls the intensity of specular highlights [S1, unofficial].
- `lightVector`: The direction vector of the light source. Usually normalized [S1, unofficial].
- `vposition`: The position of the camera/viewpoint, used to compute the view vector [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Gradient:** If the volume gradient is zero (e.g., inside a uniform region or at boundaries with no contrast), the normal cannot be computed. The method should fall back to unlit compositing or a default normal to avoid division by zero [S1, unofficial].
- **Precision:** Accumulating opacity and color requires careful handling of floating-point precision, especially with many samples. Early ray termination at 0.99 helps mitigate precision loss from very small transmittance values [S1, unofficial].
- **Boundary Artifacts:** Gradient computation near volume boundaries may be unstable if not handled with appropriate clamping or reflection padding.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. The lighting calculation adds constant overhead per sample (gradient computation, dot products, power function).
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the volume data storage.
- **Performance:** Slower than standard compositing due to gradient computation and lighting math. The cost of gradient calculation depends on the sampling method (finite differences require multiple volume lookups).
- **Strengths:** Provides strong visual cues for surface orientation and curvature, making volumes appear more solid and less "cloudy" [S1, unofficial].
- **Limitations:** The shading is local and does not account for inter-sample light transport (in-scattering). It may produce artifacts if the gradient is noisy or if the transfer function creates discontinuities.
- **When to use:** When visual clarity of internal structures and surface-like features is prioritized over physical accuracy of light transport.
- **When not to use:** For highly translucent or fog-like volumes where surface normals are not meaningful, or when performance is critical and lighting overhead is prohibitive.

## Background
Phong Shading with Volume Rendering is a technique that adapts the classic Phong reflection model, originally developed for polygonal surfaces, to volumetric data. It was implemented in educational and research contexts for direct volume ray casting, such as in the CSE-333 Computer Graphics course projects [S1, unofficial]. The method bridges the gap between surface rendering and volume rendering by treating volume samples as if they were small surface patches with normals derived from the scalar field's gradient.

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
