# Phong Shading with Volume Rendering

Also called: compositeShaderPhong

## Overview
Phong Shading with Volume Rendering enhances standard direct volume ray casting by applying Phong lighting models (diffuse and specular) to volume samples based on computed surface normals derived from volume gradients [S1, unofficial]. Unlike standard compositing which treats volume samples as opaque or translucent surfaces without geometric shading, this method calculates lighting contributions per sample to add surface-like shading and specular highlights to the volume data [S1, unofficial]. It builds on [Lighting-Specific Compositing](5940-lighting-specific-compositing.md) by defining specific formulas for $\Delta C_i$ and $\Delta \alpha_i$ using gradient-based normals.

## How it works
**Inputs:**
*   Ray origin $O$ and direction $D$.
*   Volume dataset with trilinear interpolation support for color $c$ and gradient $\nabla c$.
*   Camera position $v_{pos}$ (viewpoint).
*   Light direction vector $L_{vec}$ (assumed directional light).
*   Parameters: shininess exponent $k_s$ (often denoted as $n$ or $shininess$ in literature, here $k_s$ refers to the specular coefficient multiplier in the source, while shininess is the exponent), specular coefficient multiplier $K_{spec}$ (labeled $ks$ in source [S1, unofficial]), and ambient factor $K_{amb}$ (fixed at 0.05 in source [S1, unofficial]).
*   Accumulator state $(C_{acc}, \alpha_{acc})$ initialized to $(0, 0)$ [S1, S2, unofficial].

**Outputs:**
*   Final pixel color $C_{final}$ composited over background $C_{bg}$.

**Data Structures:**
*   Ray state: $t_0, t_1$ (entry/exit points from AABB), $sampleLength$ (step size).
*   Accumulators: $C_{acc}$ (vec3 or vec4), $\alpha_{acc}$ (float).
*   Sample data: $c$ (vec4 RGBA), $normal$ (vec3).

**Numbered Steps:**

1.  **Ray-Volume Intersection:** Perform standard Axis-Aligned Bounding Box (AABB) intersection to determine the entry point $t_0$ and exit point $t_1$ of the ray within the volume. Calculate the step size $sampleLength$ based on the desired number of samples or voxel resolution [S1, unofficial].
2.  **Initialization:** Initialize $C_{acc} = (0, 0, 0)$ and $\alpha_{acc} = 0.0$ [S1, unofficial].
3.  **Ray Casting Loop:** Iterate for $i = 0$ to $SAMPLES-1$ while the ray is within the volume bounds ($t < t_1$) and accumulated opacity $\alpha_{acc} < 0.99$ [S1, unofficial].
    a.  **Sample Retrieval:** Calculate the current ray position $P = O + t \cdot D$. Retrieve the interpolated color $c$ (RGBA) and the gradient vector $\nabla c$ at $P$ from the volume dataset [S1, unofficial].
    b.  **Normal Computation:** Set $normal = \nabla c$. If the magnitude of $normal$ is zero (or below a threshold), skip lighting calculations for this sample (treat as unlit or use default ambient) [S1, unofficial]. Otherwise, normalize $normal = \frac{normal}{\|normal\|}$ [S1, unofficial].
    c.  **Vector Calculation:** Compute the view direction vector $V = \frac{v_{pos} - P}{\|v_{pos} - P\|}$ (normalized) [S1, unofficial]. The light direction $L$ is given by $L_{vec}$ (normalized) [S1, unofficial].
    d.  **Lighting Calculation:**
        i.  Compute the diffuse term: $diffuse = \max(0, normal \cdot L)$ [S1, unofficial].
        ii. Compute the reflection vector $R = 2(n \cdot L)n - L$ [S1, unofficial].
        iii. Compute the specular term: $specular = \max(0, R \cdot V)^{shininess}$ [S1, unofficial].
        iv. Compute the modified sample color $c_{lit}$:
            $$ c_{lit} = c \cdot K_{amb} + c \cdot diffuse + c \cdot K_{spec} \cdot specular $$
            Where $K_{amb} = 0.05$ and $K_{spec}$ is the specular coefficient multiplier $ks$ from the source [S1, unofficial]. Note: The source describes combining "ambient-like term (0.05*c), diffuse term, and specular term scaled by ks". This implies the base color $c$ is modulated by these factors. A precise interpretation of "modify the sample color c by combining..." suggests:
            $$ c_{lit} = c \cdot (0.05 + diffuse + ks \cdot specular) $$
            However, standard Phong is $K_a I_a + K_d I_d (N \cdot L) + K_s I_s (R \cdot V)^n$. The source simplifies this to modifying $c$. We follow the source's explicit instruction: "modify the sample color c by combining the diffuse term, an ambient-like term (0.05*c), and the specular term scaled by ks". This results in:
            $$ c_{lit} = 0.05 \cdot c + c \cdot diffuse + c \cdot ks \cdot specular $$
            $$ c_{lit} = c \cdot (0.05 + diffuse + ks \cdot specular) $$
    e.  **Compositing:**
        i.  Extract the original alpha value from the sample: $tempa = c.a$ [S1, unofficial].
        ii. The lighting-modified color contribution is $\Delta C_i = c_{lit} \cdot tempa$ (assuming the color $c$ includes the alpha, or $c_{lit}$ is the RGB part and alpha is separate; standard volume rendering uses $c$ as RGBA, so $\Delta C_i$ is the RGB part of $c_{lit}$ scaled by alpha). The source says "apply the modified color c to the front-to-back compositing formula". In standard compositing, $\Delta C_i = c_i \alpha_i$. Here, $c_i$ is replaced by $c_{lit}$. So:
            $$ \Delta C_i = c_{lit}.rgb \cdot tempa $$
            $$ \Delta \alpha_i = tempa $$
        iii. Update accumulators:
            $$ C_{acc} = C_{acc} + (1 - \alpha_{acc}) \cdot \Delta C_i $$
            $$ \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \cdot \Delta \alpha_i $$
    f.  **Early Termination Check:** If $\alpha_{acc} \ge 0.99$, set $\alpha_{acc} = 1.0$ and break the loop [S1, unofficial].
    g.  **Advance Ray:** $t = t + sampleLength$ [S1, unofficial].
4.  **Final Compositing:** Composite the accumulated color over the background $C_{bg}$:
    $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading_volume_casting(ray_origin, ray_direction, volume_data, camera_pos, light_dir, shininess, ks):
    # 1. AABB Intersection
    t0, t1 = intersect_aabb(ray_origin, ray_direction, volume_data.bbox)
    if t0 > t1:
        return background_color # No hit
    
    sample_length = (t1 - t0) / num_samples
    t = t0
    
    # 2. Initialization
    c_acc = vec3(0.0, 0.0, 0.0)
    alpha_acc = 0.0
    
    # Pre-normalize light direction
    l_vec = normalize(light_dir)
    
    for i in range(num_samples):
        # Check bounds and early termination
        if t > t1 or alpha_acc >= 0.99:
            break
            
        # 3a. Sample Retrieval
        pos = ray_origin + t * ray_direction
        sample_rgba = volume_data.interpolate_rgba(pos)
        gradient = volume_data.interpolate_gradient(pos)
        
        c = sample_rgba.rgb
        alpha = sample_rgba.a
        
        # 3b. Normal Computation
        normal = gradient
        if length(normal) < 1e-6:
            # No normal, use ambient only or skip lighting
            diffuse = 0.0
            specular = 0.0
        else:
            normal = normalize(normal)
            
            # 3c. Vector Calculation
            view_dir = normalize(camera_pos - pos)
            
            # 3d. Lighting Calculation
            # Diffuse
            dot_nl = max(0.0, dot(normal, l_vec))
            diffuse = dot_nl
            
            # Specular
            # Reflection vector R = 2(N.L)N - L
            r_vec = 2.0 * dot_nl * normal - l_vec
            dot_rv = max(0.0, dot(r_vec, view_dir))
            specular = pow(dot_rv, shininess)
            
        # 3d. Modify Color
        # Source: "modify the sample color c by combining the diffuse term, 
        # an ambient-like term (0.05*c), and the specular term scaled by ks"
        # Formula: c_lit = c * (0.05 + diffuse + ks * specular)
        lighting_factor = 0.05 + diffuse + ks * specular
        c_lit = c * lighting_factor
        
        # 3e. Compositing
        # Delta C is the lighting-modified color contribution
        delta_c = c_lit * alpha
        delta_alpha = alpha
        
        # Front-to-back compositing
        transmittance = 1.0 - alpha_acc
        c_acc = c_acc + transmittance * delta_c
        alpha_acc = alpha_acc + transmittance * delta_alpha
        
        # 3f. Early Termination
        if alpha_acc >= 0.99:
            alpha_acc = 1.0
            break
            
        # 3g. Advance
        t += sample_length
        
    # 4. Final Compositing
    final_color = c_acc + (1.0 - alpha_acc) * background_color
    return final_color
```

**Parameters:**
*   `shininess`: The specular exponent for the Phong model. Typical values range from 1 to 1000, depending on the desired glossiness [S1, unofficial].
*   `ks` (or $K_{spec}$): The specular coefficient multiplier. Controls the intensity of the specular highlight [S1, unofficial].
*   `lightVector`: The direction vector of the light source. Assumed to be a directional light in the source implementation [S1, unofficial].
*   `vposition`: The position of the camera/viewpoint, used to compute the view vector for specular reflection [S1, unofficial].
*   Ambient factor: Fixed at 0.05 in the source implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If the gradient magnitude is zero (e.g., in uniform regions), the normal cannot be computed. The source implies skipping lighting or using a default. In implementation, this results in no diffuse/specular contribution, only ambient [S1, unofficial].
*   **Precision:** Normalization of gradients and vectors should handle near-zero magnitudes to avoid division by zero [G].
*   **Boundary:** The AABB intersection must correctly handle rays that miss the volume entirely [S1, unofficial].
*   **Alpha Threshold:** The early termination threshold of 0.99 is specific to the source implementation. Other implementations may use different thresholds [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. The lighting calculation adds constant overhead per sample (gradient lookup, vector math).
*   **Memory Complexity:** $O(1)$ additional memory per pixel for accumulators. The volume data itself is the primary memory consumer.
*   **Performance:** The method is computationally more expensive than standard compositing due to gradient computation and per-sample lighting calculations. However, it provides significantly better visual quality by adding surface-like shading.
*   **Strengths:** Enhances the perception of surface details and depth in volume data by simulating surface lighting. Simple to implement as it integrates directly into the standard ray casting loop.
*   **Limitations:** Relies on accurate gradient estimation; noisy gradients can lead to noisy lighting. Does not account for light transport effects like subsurface scattering or in-scattering. The specular highlights are view-dependent and may not accurately represent the material properties of the volume.
*   **When to use:** When visual clarity of internal structures and surface-like shading is desired, and computational resources allow for per-sample lighting calculations.
*   **When not to use:** When real-time performance is critical and the volume data is large, or when physically accurate light transport is required.

## Background
Phong Shading with Volume Rendering is a technique described in the context of a computer graphics course project (CSE-333) at the University of Washington, implemented in C++ using GLM [S1, unofficial]. It extends the standard composite shader by incorporating gradient-based normals and Phong lighting calculations [S1, unofficial]. The technique is part of a family of lighting-specific compositing methods aimed at improving the visual quality of volume rendering by adding geometric shading cues.

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
