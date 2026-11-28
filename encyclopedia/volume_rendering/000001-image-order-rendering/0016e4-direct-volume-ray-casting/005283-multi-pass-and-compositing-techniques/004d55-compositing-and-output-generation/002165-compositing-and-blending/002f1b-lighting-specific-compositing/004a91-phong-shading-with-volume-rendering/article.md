# Phong Shading with Volume Rendering
Also called: compositeShaderPhong

## Overview
Phong Shading with Volume Rendering enhances standard direct volume ray casting by applying Phong lighting (diffuse and specular) to volume samples based on surface normals derived from volume gradients [S1, unofficial]. Unlike standard compositing which treats samples as colored blobs, this method calculates surface-like shading interactions before compositing, adding geometric depth cues to the volume [S1, unofficial]. It differs from standard composite shaders by incorporating gradient-based normals and Phong lighting calculations into the color computation before compositing [S1, unofficial].

## How it works
**Inputs:**
- Volume dataset (scalar field).
- Transfer function mapping scalar values to RGBA (color $c$, opacity $\alpha$).
- Camera position ($v_{position}$).
- Light direction vector ($lightVector$).
- Phong parameters: shininess exponent, specular coefficient ($k_s$).
- Ray origin and direction.
- Volume bounding box (AABB).
- Number of samples per ray.
- Background color.

**Outputs:**
- Final pixel color $C_{final}$.

**Data Structures:**
- Sample point: 3D position $(x, y, z)$.
- Sample color: RGB vector $c$.
- Sample opacity: float $\alpha$.
- Gradient: 3D vector $\nabla V$ (computed via finite differences).

**Steps:**
1. **Ray-Volume Intersection:** Perform standard Axis-Aligned Bounding Box (AABB) intersection to determine the entry parameter $t_0$ and exit parameter $t_1$ [S1, unofficial]. Calculate the step size (sample length) as $(t_1 - t_0) / \text{num\_samples}$.
2. **Initialization:** Initialize the color accumulator $C_{acc} = (0, 0, 0)$, accumulated opacity $\alpha_{acc} = 0.0$, and transmittance $T = 1.0$ [G].
3. **Ray Casting Loop:** Iterate $i$ from $0$ to $\text{num\_samples}-1$ while the ray remains within the volume bounds and $\alpha_{acc} < 0.99$ [S1, unofficial].
    a. **Sample Retrieval:** At the current ray position, retrieve the interpolated color $c$ (RGB) and opacity $\alpha$ from the transfer function [S1, unofficial].
    b. **Normal Computation:** Retrieve the gradient vector $\mathbf{n}$ at the current ray position [S1, unofficial]. If $\|\mathbf{n}\| > 0$, normalize it to obtain the surface normal $\mathbf{N} = \mathbf{n} / \|\mathbf{n}\|$ [S1, unofficial]. If the gradient is zero, skip lighting calculations and use the unmodified sample color.
    c. **Lighting Calculation:**
        i. Compute the view direction vector $\mathbf{V}$ from the sample point to the camera position ($v_{position}$) and normalize it [S1, unofficial].
        ii. Normalize the light direction vector $\mathbf{L} = lightVector$ [S1, unofficial].
        iii. Compute the diffuse term as the clamped dot product of the normal and the light vector: $D = \max(0, \mathbf{N} \cdot \mathbf{L})$ [S1, unofficial].
        iv. Compute the reflection vector $\mathbf{R} = 2(\mathbf{N} \cdot \mathbf{L})\mathbf{N} - \mathbf{L}$ [S1, unofficial].
        v. Compute the specular term as the shininess power of the clamped dot product of the reflected light vector and the view direction, scaled by $k_s$: $S = k_s \cdot (\max(0, \mathbf{R} \cdot \mathbf{V}))^{\text{shininess}}$ [S1, unofficial].
        vi. Modify the sample color $c$ by combining the diffuse term, an ambient-like term ($0.05 \cdot c$), and the specular term: $c' = c \cdot (0.05 + D) + S$ [S1, unofficial].
    d. **Compositing:** Apply the modified color $c'$ and the original opacity $\alpha$ to the front-to-back compositing equation:
       $$ C_{acc} \leftarrow C_{acc} + T \cdot \alpha \cdot c' $$
       $$ \alpha_{acc} \leftarrow \alpha_{acc} + T \cdot \alpha $$
       $$ T \leftarrow T \cdot (1 - \alpha) $$
    e. **Termination Check:** If $\alpha_{acc} \ge 0.99$, set $\alpha_{acc} = 1.0$ and break the loop [S1, unofficial].
    f. **Advance:** Move the ray position forward by the sample length [S1, unofficial].
4. **Background Composite:** Composite the result over the background color $C_{bg}$:
   $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_volume_shading(ray_origin, ray_direction, volume_bounds, num_samples, tf_control_points, camera_pos, light_dir, shininess, ks, background_color):
    # 1. AABB Intersection (slab method)
    t0, t1 = 0.0, 0.0
    min_t, max_t = 0.0, 1e9
    for axis in range(3):
        if abs(ray_direction[axis]) < 1e-8:
            if ray_origin[axis] < volume_bounds[axis*2] or ray_origin[axis] > volume_bounds[axis*2+1]:
                return background_color
        else:
            t_entry = (volume_bounds[axis*2] - ray_origin[axis]) / ray_direction[axis]
            t_exit = (volume_bounds[axis*2+1] - ray_origin[axis]) / ray_direction[axis]
            if t_entry > t_exit: t_entry, t_exit = t_exit, t_entry
            if t_entry > min_t: min_t = t_entry
            if t_exit < max_t: max_t = t_exit
    t0, t1 = min_t, max_t
    if t1 <= t0:
        return background_color

    step_size = (t1 - t0) / num_samples

    # 2. Initialization
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    T = 1.0

    # 3. Ray Casting Loop
    for i in range(num_samples):
        t = t0 + i * step_size
        pos = ray_origin + t * ray_direction

        # a. Sample Retrieval (standard trilinear interpolation + TF lookup) [G]
        scalar = sample_trilinear(pos, volume_data)
        c_i, alpha_i = lookup_transfer_function(scalar, tf_control_points)

        # b. Normal Computation (central differences) [G]
        h = step_size * 0.5
        grad = vec3(
            sample_trilinear(pos + vec3(h, 0, 0), volume_data) - sample_trilinear(pos - vec3(h, 0, 0), volume_data),
            sample_trilinear(pos + vec3(0, h, 0), volume_data) - sample_trilinear(pos - vec3(0, h, 0), volume_data),
            sample_trilinear(pos + vec3(0, 0, h), volume_data) - sample_trilinear(pos - vec3(0, 0, h), volume_data)
        ) / (2.0 * h)
        grad_norm = length(grad)

        if grad_norm > 1e-6:
            N = normalize(grad)
            V = normalize(camera_pos - pos)
            L = normalize(light_dir)

            # c. Lighting Calculation
            diff = max(0.0, dot(N, L))
            R = 2.0 * dot(N, L) * N - L
            spec = ks * pow(max(0.0, dot(R, V)), shininess)
            c_i = c_i * (0.05 + diff) + spec

        # d. Compositing
        C_acc = C_acc + T * alpha_i * c_i
        alpha_acc = alpha_acc + T * alpha_i
        T = T * (1.0 - alpha_i)

        # e. Termination Check
        if alpha_acc >= 0.99:
            alpha_acc = 1.0
            break

    # 4. Background Composite
    return C_acc + (1.0 - alpha_acc) * background_color
```

**Parameters:**
- `shininess`: The specular exponent for the Phong model [S1, unofficial].
- `ks`: The specular coefficient multiplier [S1, unofficial].
- `lightVector`: The direction vector of the light source [S1, unofficial].
- `vposition`: The position of the camera/viewpoint, used to compute the view vector [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Gradient:** If the volume gradient is zero (e.g., inside a uniform region), the normal cannot be computed. The method skips lighting calculations for that sample and uses the unmodified sample color [S1, unofficial].
- **Precision:** Accumulating opacity and color requires careful handling of floating-point precision. Early ray termination at 0.99 stops the loop when further samples contribute negligibly [G].
- **Boundary Artifacts:** Gradient computation near volume boundaries may be unstable if not handled with appropriate clamping or reflection padding [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. The lighting calculation adds constant overhead per sample (gradient computation, dot products, power function) [G].
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the volume data storage [G].
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Phong Shading with Volume Rendering is a technique that adapts the classic Phong reflection model to volumetric data by treating volume samples as small surface patches with normals derived from the scalar field's gradient [S1, unofficial]. It was developed as part of direct volume ray casting implementations [S1, unofficial].

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
