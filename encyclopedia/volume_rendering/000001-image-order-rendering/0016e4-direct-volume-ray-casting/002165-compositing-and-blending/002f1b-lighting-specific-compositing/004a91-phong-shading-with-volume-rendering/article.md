# Phong Shading with Volume Rendering
Also called: compositeShaderPhong

## Overview
Phong Shading with Volume Rendering is a lighting-specific compositing technique that enhances standard direct volume ray casting by applying Phong illumination models (diffuse and specular) to volume samples [S1, unofficial]. It differs from the standard composite shader by incorporating gradient-based surface normals and lighting calculations into the color contribution before compositing, thereby adding surface-like shading cues to the volumetric data [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset (scalar or RGBA), a transfer function mapping scalar values to color/opacity, camera position ($v_{position}$), light direction vector ($lightVector$), and lighting parameters (shininess, specular coefficient $k_s$).
*   **Output:** A 2D image with Phong-shaded volumetric appearance.

**Data Structures**
The method operates on the standard ray-casting accumulator state $(C_{acc}, \alpha_{acc})$ [S1, S51, unofficial].
*   $C_{acc}$: Accumulated color (vec3 or vec4).
*   $\alpha_{acc}$: Accumulated opacity (float).
*   Sample data: Interpolated color $c$ (RGBA) and gradient vector $normal$ (vec3) retrieved from the volume at the current ray position.

**Numbered Steps**
1.  **Ray Setup:** Perform standard Axis-Aligned Bounding Box (AABB) intersection to determine the entry point $t_0$, exit point $t_1$, and step size $sampleLength$ [S1, unofficial].
2.  **Initialization:** Initialize $C_{acc} = (0, 0, 0)$ and $\alpha_{acc} = 0.0$ [S1, unofficial]. Initialize transmittance $T = 1.0$ [G].
3.  **Ray Marching Loop:** Iterate for $i$ from $0$ to $SAMPLES-1$ while the ray is within the volume bounds and $\alpha_{acc} < 0.99$ [S1, unofficial]:
    a.  **Sample Retrieval:** Retrieve the interpolated color $c$ and the gradient vector $normal$ at the current ray position [S1, unofficial].
    b.  **Normal Processing:** If $normal$ is not zero, normalize it to obtain $\hat{n}$ [S1, unofficial]. If $normal$ is zero, skip lighting calculation for this sample (or use a default normal, though the source implies skipping/modifying $c$ only if normal exists) [S1, unofficial].
    c.  **Vector Computation:** Compute the view direction vector $\hat{v}$ from the sample point to the camera position ($v_{position}$) and normalize it [S1, unofficial]. Normalize the light direction vector $\hat{l} = lightVector$ [S1, unofficial].
    d.  **Diffuse Term:** Compute the diffuse term $K_d$ as the clamped dot product of the normal and light vector:
        $$ K_d = \max(0, \hat{n} \cdot \hat{l}) $$
    e.  **Specular Term:** Compute the reflection vector $\hat{r}$ of the light vector about the normal:
        $$ \hat{r} = 2(\hat{n} \cdot \hat{l})\hat{n} - \hat{l} $$
        Compute the specular term $K_s$ as the shininess power of the clamped dot product of the reflected light vector and the view direction:
        $$ K_s = \max(0, \hat{r} \cdot \hat{v})^{shininess} $$
    f.  **Color Modification:** Modify the sample color $c$ by combining the diffuse term, an ambient-like term ($0.05 \cdot c$), and the specular term scaled by $k_s$. The source specifies:
        $$ c_{lit} = c \cdot (0.05 + K_d) + c \cdot k_s \cdot K_s $$
        *(Note: The source text says "modify the sample color c by combining the diffuse term, an ambient-like term (0.05*c), and the specular term scaled by ks". Standard Phong is $K_a I_a + K_d I_d (\hat{n}\cdot\hat{l}) + K_s I_s (\hat{r}\cdot\hat{v})$. The source implies the base color $c$ is modulated by the lighting terms. A precise interpretation of "0.05*c" as ambient and "diffuse term" as the dot product suggests: $c_{lit} = c \cdot 0.05 + c \cdot K_d + c \cdot k_s \cdot K_s$. However, "diffuse term" usually includes the light intensity. Assuming unit light intensity and color $c$ represents the base material color, the formula is likely: $c_{lit} = c \cdot (0.05 + K_d + k_s \cdot K_s)$ or similar. Given the ambiguity, we stick to the explicit components mentioned: ambient $0.05c$, diffuse $K_d$, specular $k_s K_s$. The most standard implementation of "combining" these with the base color $c$ is: $c_{lit} = c \cdot 0.05 + c \cdot K_d + c \cdot k_s \cdot K_s$.)*
        *Correction based on strict source text:* "compute the diffuse term as the clamped dot product... compute the specular term... modify the sample color c by combining the diffuse term, an ambient-like term (0.05*c), and the specular term scaled by ks".
        Let $diffuse = \max(0, \hat{n} \cdot \hat{l})$.
        Let $specular = \max(0, \hat{r} \cdot \hat{v})^{shininess}$.
        $c_{lit} = 0.05 \cdot c + c \cdot diffuse + c \cdot k_s \cdot specular$.
        *(This assumes the light intensity is 1 and the color $c$ is the base color. If $c$ already includes lighting, this would be double lighting. In volume rendering, $c$ is usually the transfer function color. So this adds Phong shading to the TF color.)*
    g.  **Compositing:** Apply the modified color $c_{lit}$ to the front-to-back compositing formula. The effective opacity contribution $\Delta \alpha_i$ is the original alpha value from the sample (preserved in $tempa$) [S1, unofficial]. The effective color contribution $\Delta C_i$ is $c_{lit}$ [S1, unofficial].
        $$ C_{acc} \leftarrow C_{acc} + c_{lit} \cdot (1 - \alpha_{acc}) $$
        $$ \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_{sample} $$
    h.  **Termination Check:** If $\alpha_{acc} \ge 0.99$, set $\alpha_{acc} = 1.0$ and break the loop [S1, unofficial].
    i.  **Advance:** Advance the ray position by $sampleLength$ [S1, unofficial].
4.  **Final Compositing:** Composite the accumulated color over the background $C_{bg}$:
    $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$
    [S1, S14, unofficial].

**Parameters**
*   $shininess$: The specular exponent for the Phong model [S1, unofficial]. Typical values range from 1 to 1000, depending on desired glossiness.
*   $k_s$: The specular coefficient multiplier [S1, unofficial].
*   $lightVector$: The direction vector of the light source [S1, unofficial].
*   $v_{position}$: The position of the camera/viewpoint [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Zero Gradient:** If the gradient vector is zero (e.g., in homogeneous regions), the normal cannot be computed. The source implies skipping the lighting modification or using a default, but does not specify a fallback. Implementers should handle this to avoid division by zero or invalid normals [S1, unofficial].
*   **Precision:** Accumulating opacity and color requires careful handling of floating-point precision, especially with many samples. Early ray termination at 0.99 helps mitigate precision issues and performance costs [S1, unofficial].
*   **Boundary:** The AABB intersection must be robust to handle rays that miss the volume entirely [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples per ray. The lighting calculation (dot products, power) adds a constant overhead per sample compared to standard compositing.
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(1)$ per ray for accumulators.
*   **Performance:** The additional gradient computation and lighting math increase per-sample cost. However, early ray termination remains applicable, limiting the average sample count [S1, S10, S12, unofficial].
*   **Strengths:** Provides intuitive surface-like shading cues, enhancing the perception of shape and depth in volumetric data [S1, unofficial].
*   **Limitations:** Relies on accurate gradient estimation; noisy data can produce noisy normals and artifacts. The Phong model is a surface approximation and may not accurately represent complex internal light transport (e.g., subsurface scattering) [S1, unofficial].
*   **When to use:** When visual clarity of internal structures and boundaries is prioritized over physical accuracy of light transport.
*   **When not to use:** When physically accurate volumetric lighting (e.g., in-scattering) is required, or when the data is too noisy for stable gradient estimation.

## Background
Phong Shading with Volume Rendering is a technique described in the context of computer graphics education and implementation, specifically within the CSE-333 Computer Graphics course materials [S1, unofficial]. It extends the basic volume ray casting algorithm by integrating the classic Phong reflection model, which was introduced by Bui Tuong Phong in 1975 for surface shading, into the volume rendering pipeline [S1, unofficial]. This approach allows for the visualization of volumetric data with enhanced surface-like properties, bridging the gap between surface rendering and volume rendering techniques.

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
