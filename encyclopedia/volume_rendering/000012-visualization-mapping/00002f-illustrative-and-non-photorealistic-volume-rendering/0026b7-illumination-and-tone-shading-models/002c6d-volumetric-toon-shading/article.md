# Volumetric Toon Shading

Also called: Fragment Shader Toon-Shading for Volumes, Interactive volume cartoon shading, Toon-Shading for Volumes, Toon-shading via 3D normal map

## Overview
Volumetric Toon Shading is a non-photorealistic rendering technique that applies cel-shading effects to volumetric data sets without requiring explicit surface vertices or mesh extraction [S1, unofficial]. It differs from polygon-based toon shading by computing lighting per-fragment using a pre-computed 3D normal map and hardware texture combiners, allowing for interactive rendering of direct volume rendering results [S1, unofficial]. Unlike [Lit Sphere Shading](785-lit-sphere-shading.md) which approximates lighting on implicit surfaces, this method operates directly on the voxel gradient field.

## How it works
**Inputs:**
- A 3D scalar volume dataset.
- A pre-computed 3D normal map texture storing gray-scale gradients in RGB components.
- A parallel light direction vector $L$.
- A 1D toon-shading texture (lookup table).
- Two user-specified tone values (registers) for illuminated and shadowed regions.
- A threshold value (default 0.5) and an optional offset.

**Outputs:**
- A 2D image with discrete tonal shading (cel-shaded appearance).

**Data Structures:**
1. **3D Normal Map:** A 3D texture where each texel $(x, y, z)$ stores the gradient vector $\nabla V$ at that voxel. The components are stored as gray-scale values in the Red, Green, and Blue channels. Let $N(x,y,z)$ be the normal vector retrieved from this texture.
2. **Toon Texture:** A 1D texture used to map dot product values to final colors. Nearest-neighbor interpolation is required to create sharp discontinuities.

**Steps:**
1. **Render Normal Map:** Render the 3D normal map into a framebuffer. This step establishes the per-fragment normal vectors $N$ for the volume. [S1, unofficial]
2. **Issue Light Vector:** Issue the parallel light direction vector $L$ as a constant color input to the texture combiner stage. [S1, unofficial]
3. **Compute Dot Product:** For each fragment, retrieve the normal $N$ from the 3D normal map. Compute the dot product between the light vector $L$ and the normal $N$:
   $$ D = L \cdot N $$
   This computation is performed using per-fragment arithmetic in the fragment shader or hardware combiner. [S1, unofficial]
4. **Apply Offset (Optional):** Add an offset $O$ to the dot product value to shift the shading threshold:
   $$ D' = D + O $$
   [S1, unofficial]
5. **Thresholding via Multiplexer:** Use the resulting value $D'$ to trigger a multiplexer in the texture combiner stage. The multiplexer compares $D'$ against a threshold $T$ (default $T = 0.5$).
   - If $D' > T$, select Register 1 (Illuminated Tone).
   - If $D' \leq T$, select Register 2 (Shadow Tone).
   [S1, unofficial]
6. **Lookup Color:** Use the selected register's value as a 1D texture coordinate to look up the final color in the toon-shading texture. Alternatively, the registers themselves may hold the final color values directly if the hardware supports direct register output. The sources describe writing user-specified tones to the two registers selected by the multiplexer. [S1, unofficial]
7. **Interpolation:** Ensure nearest-neighbor texture interpolation is enabled for the toon texture lookup to maintain sharp boundaries between tones. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Implementation
# Inputs:
#   normal_map: 3D texture storing gradients in RGB
#   toon_tex: 1D texture for color lookup
#   light_dir: vec3, normalized light direction
#   threshold: float, default 0.5
#   offset: float, default 0.0
#   tone_lit: vec3, color for illuminated regions
#   tone_shad: vec3, color for shadowed regions

void main() {
    // 1. Get fragment position in volume space (e.g., from raycast or screen quad)
    vec3 frag_pos = ...; // Implementation depends on volume rendering pass
    
    // 2. Sample normal from 3D normal map
    vec3 N = texture(normal_map, frag_pos).rgb;
    
    // 3. Compute dot product
    float dot_product = dot(light_dir, N);
    
    // 4. Apply offset
    float adjusted_dot = dot_product + offset;
    
    // 5. Thresholding
    if (adjusted_dot > threshold) {
        // Illuminated region
        gl_FragColor = vec4(tone_lit, 1.0);
    } else {
        // Shadowed region
        gl_FragColor = vec4(tone_shad, 1.0);
    }
}
```
Note: The original method relies on hardware texture combiners and multiplexers. The pseudocode above translates this logic into a standard fragment shader for clarity. The "registers" mentioned in the source are effectively the conditional branches selecting between the two tone values. [S1, unofficial]

**Parameters:**
- **Threshold:** 0.5. Separates illuminated points from shadow points. [S1, unofficial]
- **Offset:** Arbitrary value to shift the threshold. [S1, unofficial]
- **Tones:** Two user-defined colors for light and shadow. [S1, unofficial]

**Edge Cases:**
- **Zero Gradient:** If the gradient is zero, the normal $N$ is undefined. [G] Handle by setting $N$ to a default value (e.g., $(0,0,1)$) or skipping shading.
- **Boundary Conditions:** The 3D normal map must be correctly sampled at volume boundaries. [G] Use appropriate texture wrapping or clamp-to-edge.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of fragments rendered. The per-fragment cost is constant (dot product, comparison, texture lookup).
- **Memory Complexity:** Requires storage for the 3D normal map (3 components per voxel) and the 1D toon texture.
- **Performance:** Enables interactive rendering speeds because it avoids expensive ray-marching for shading calculations and leverages hardware texture combiners. [S1, unofficial]
- **Strengths:** No need for explicit surface extraction or mesh generation. Simple to implement on hardware supporting multitexturing and fragment shaders. [S1, unofficial]
- **Limitations:** Shading is based on volume gradients, not surface normals, which may not always correspond to perceived visual edges. Relies on hardware features (texture combiners) which may be deprecated in modern APIs. [S1, unofficial]
- **When to use:** When interactive performance is required for non-photorealistic volume visualization and explicit geometry is not available or desirable.

## Background
The method was introduced in the context of interactive volume illustration. It adapts the concept of toon-shading polygonal objects, where the dot product $L \cdot N$ is used as a texture coordinate, to volumetric data by using a 3D normal map instead of vertex normals. [S1, unofficial] The approach was developed to allow integration with hatch-based approaches and direct volume rendering algorithms via 3D texture maps. [S1, unofficial]

## Sources
[S1] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
