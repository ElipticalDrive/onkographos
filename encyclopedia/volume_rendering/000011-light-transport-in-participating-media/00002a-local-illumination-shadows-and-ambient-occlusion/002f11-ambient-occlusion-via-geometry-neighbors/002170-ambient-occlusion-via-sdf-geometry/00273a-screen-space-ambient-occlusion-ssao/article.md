# Screen-Space Ambient Occlusion (SSAO)
Also called: Crytek's Screen-Space AO, SSAO (Original), Screen-Space Ambient Occlusion (SSAO) Pass

## Overview
Screen-Space Ambient Occlusion (SSAO) is a post-processing technique that estimates ambient occlusion by sampling points in a hemisphere around each pixel's surface position and checking if those points are occluded by geometry in the depth buffer [S1, S2, unofficial]. It is the first screen-space method for ambient occlusion, using only the Z-Buffer to compute AO at full resolution [S2, unofficial]. Unlike SDF-based methods which rely on global geometric representations, SSAO relies exclusively on screen-space data (depth and normals), meaning its performance cost is independent of scene complexity and depends only on output resolution [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `hitPosTex`: Texture containing the world-space position of the fragment hit by the primary ray.
    *   `hitNormTex`: Texture containing the world-space normal of the fragment.
    *   `depthTex`: Texture containing the depth buffer information and cell size parameters.
    *   `noiseTex`: Texture containing random vectors used for tangent space generation to break up rotational symmetry.
    *   `projMat`: Projection matrix.
    *   `viewMat`: View matrix.
    *   `radius`: The radius of the hemisphere used for sampling, determining the scale of occlusion.
    *   `NUM_SAMPLES`: The number of random samples taken per pixel for the occlusion estimation.
*   **Output:**
    *   `outVisibility`: A float texture at location 0, storing the final AO factor (1.0 = fully lit, 0.0 = fully occluded).

**Data Structures:**
*   **TBN Matrix:** A 3x3 matrix constructed from the tangent, bitangent, and normal vectors, used to rotate sample vectors from tangent space to world space.
*   **Sample Vectors:** A pre-generated array of `NUM_SAMPLES` random vectors in tangent space, typically distributed on a hemisphere or sphere.

**Steps:**
1.  **Fetch Surface Data:** For the current fragment coordinate, fetch the hit position and normal from their respective textures.
    $$ \mathbf{p} = \text{texelFetch}(\text{hitPosTex}, \text{coord}, 0).xyz $$
    $$ \mathbf{n} = \text{texelFetch}(\text{hitNormTex}, \text{coord}, 0).xyz $$
2.  **Discard Background:** If the hit position is zero (indicating no geometry was hit), discard the fragment.
    ```python
    if hitPos == vec3(0, 0, 0):
        discard
    ```
3.  **Offset Position:** Offset the hit position slightly along the normal to avoid self-intersection or z-fighting. The offset amount is derived from the depth cell size.
    $$ \mathbf{p}_{offset} = \mathbf{p} + \mathbf{n} \times \frac{\text{hitDepth\_cellSize}.y}{5.0} $$
    [G] The specific divisor (5.0) and cell size usage are implementation-specific details from the source code to ensure the sample starts just outside the surface.
4.  **Construct TBN Matrix:** Generate a tangent vector using a random vector from `noiseTex` and the normal, then compute the bitangent to form the TBN matrix.
    $$ \mathbf{t} = \text{normalize}(\mathbf{r} - \mathbf{n} \times (\mathbf{r} \cdot \mathbf{n})) $$
    $$ \mathbf{b} = \mathbf{n} \times \mathbf{t} $$
    $$ \text{TBN} = [\mathbf{t}, \mathbf{b}, \mathbf{n}] $$
    where $\mathbf{r}$ is the random vector fetched from `noiseTex`.
5.  **Sample Hemisphere:** Loop through `NUM_SAMPLES`. For each sample index $i$:
    a. Retrieve the sample vector $\mathbf{s}_{tan}$ from the pre-generated sample array.
    b. Transform $\mathbf{s}_{tan}$ into world space using the TBN matrix and scale by `radius`.
    $$ \mathbf{s}_{world} = \text{TBN} \times \mathbf{s}_{tan} \times \text{radius} $$
    c. Compute the sample position in world space.
    $$ \mathbf{p}_{samp} = \mathbf{p}_{offset} + \mathbf{s}_{world} $$
6.  **Project to Screen Space:** Project the sample position $\mathbf{p}_{samp}$ into clip space and then to normalized device coordinates (NDC).
    $$ \mathbf{v}_{clip} = \text{projMat} \times \text{viewMat} \times [\mathbf{p}_{samp}, 1.0]^T $$
    $$ \mathbf{v}_{ndc} = \frac{\mathbf{v}_{clip}.xyz}{\mathbf{v}_{clip}.w} $$
    $$ \mathbf{v}_{screen} = \mathbf{v}_{ndc} \times 0.5 + 0.5 $$
7.  **Check Screen Bounds:** Verify if the projected sample lies within the screen bounds $[0, 1]$.
    ```python
    if all(lessThanEqual(v_screen.xy, vec2(1, 1))) and all(greaterThanEqual(v_screen.xy, vec2(0, 0))):
    ```
8.  **Accumulate Occlusion:** If within bounds, perform a range check and depth comparison.
    a. Fetch the depth value at the projected screen coordinate.
    $$ d_{samp} = \text{texelFetch}(\text{depthTex}, \text{screenCoord}, 0).x $$
    b. Check if the sample is within the radius range.
    $$ \text{rangeCheck} = \begin{cases} 1.0 & \text{if } |d_{hit} - d_{samp}| < \text{radius} \\ 0.0 & \text{otherwise} \end{cases} $$
    c. Check if the sample is occluded (closer to the camera than the hit point).
    $$ \text{isOccluded} = \begin{cases} 1.0 & \text{if } d_{samp} < \text{length}(\mathbf{p}_{cam} - \mathbf{p}_{samp}) \\ 0.0 & \text{otherwise} \end{cases} $$
    d. Add to occlusion accumulator.
    $$ \text{occlusion} += \text{isOccluded} \times \text{rangeCheck} $$
    [G] `d_hit` is the depth of the original hit point, and `p_cam` is the camera position. The depth comparison logic assumes linear or appropriate depth buffer mapping.
9.  **Calculate Final Visibility:** Average the occlusion count.
    $$ AO = \begin{cases} 1.0 - \frac{\text{occlusion}}{\text{count}} & \text{if } \text{count} > 0 \\ 1.0 & \text{otherwise} \end{cases} $$
    where `count` is the number of valid samples within the radius.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Pseudocode for SSAO
# Inputs: hitPosTex, hitNormTex, depthTex, noiseTex, projMat, viewMat, radius, NUM_SAMPLES
# Output: outVisibility

void main() {
    vec2 coord = gl_FragCoord.xy / resolution;
    
    // 1. Fetch surface data
    vec3 hitPos = texelFetch(hitPosTex, ivec2(coord * textureSize(hitPosTex, 0)), 0).xyz;
    vec3 hitNorm = texelFetch(hitNormTex, ivec2(coord * textureSize(hitNormTex, 0)), 0).xyz;
    
    // 2. Discard background
    if (hitPos == vec3(0.0)) {
        discard;
    }
    
    // 3. Offset position to avoid self-intersection
    float cellSizeY = texelFetch(depthTex, ivec2(coord * textureSize(depthTex, 0)), 0).y; // Assuming cell size stored in depth tex
    hitPos += hitNorm * (cellSizeY / 5.0);
    
    // 4. Construct TBN matrix
    vec3 randVec = texelFetch(noiseTex, ivec2(coord * textureSize(noiseTex, 0)), 0).xyz;
    vec3 tangent = normalize(randVec - hitNorm * dot(randVec, hitNorm));
    vec3 bitangent = cross(hitNorm, tangent);
    mat3 tbn = mat3(tangent, bitangent, hitNorm);
    
    float occlusion = 0.0;
    int count = 0;
    
    // 5. Sample hemisphere
    for (int i = 0; i < NUM_SAMPLES; i++) {
        vec3 samp = tbn * samples[i]; // samples[i] is pre-generated tangent space vector
        samp = hitPos + samp * radius;
        
        // 6. Project to screen space
        vec4 offset = vec4(samp, 1.0);
        offset = projMat * viewMat * offset;
        offset.xyz /= offset.w;
        offset.xyz = offset.xyz * 0.5 + 0.5;
        
        // 7. Check screen bounds
        if (all(lessThanEqual(offset.xy, vec2(1.0))) && all(greaterThanEqual(offset.xy, vec2(0.0)))) {
            // 8. Accumulate occlusion
            vec2 screenCoord = offset.xy * resolution;
            float sampleDepth = texelFetch(depthTex, ivec2(screenCoord), 0).x;
            
            float rangeCheck = (abs(texelFetch(depthTex, ivec2(coord * textureSize(depthTex, 0)), 0).x - sampleDepth) < radius) ? 1.0 : 0.0;
            
            // Note: length(camPos - samp) requires camera position; assuming it's available or approximated
            float isOccluded = (sampleDepth < length(cameraPos - samp)) ? 1.0 : 0.0;
            
            occlusion += isOccluded * rangeCheck;
            count++;
        }
    }
    
    // 9. Calculate final visibility
    float outVisibility = (count > 0.0) ? 1.0 - occlusion / count : 1.0;
    outVisibility = outVisibility; // Output to location 0
}
```

**Parameters:**
*   `NUM_SAMPLES`: The number of random samples taken per pixel. Higher values reduce noise but increase computational cost. Not given in the sources.
*   `radius`: The radius of the hemisphere used for sampling. Determines the scale of the occlusion effect. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Halo Effect:** Because the method is not cosine-weighted, it accounts for occlusion that should not be accounted for, leading to blackened edges around objects [S2, unofficial].
*   **Self-Intersection:** Without proper offsetting, samples may intersect the surface they are sampling from, causing incorrect occlusion values [S1, unofficial].
*   **Screen Edges:** Samples near the edge of the screen may project outside the visible area, requiring bounds checking [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of pixels and $R$ is the number of samples per pixel. The cost is independent of scene geometry complexity [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the input textures (position, normal, depth) and output texture.
*   **Performance:** Performance depends only on output resolution, not scene complexity [S2, unofficial].
*   **Strengths:** Real-time performance on modern hardware; no need for complex geometric data structures like SDFs or BVHs [S2, unofficial].
*   **Limitations:** Limited to visible geometry (screen-space); suffers from the halo effect due to lack of cosine weighting [S2, unofficial]; artifacts at screen edges and near depth discontinuities.

## Background
SSAO was introduced by Crytek as the first screen-space method for ambient occlusion, utilizing only the Z-Buffer to compute AO at full resolution [S2, unofficial]. It was later expanded upon by Loos and Sloan, who formulated it as a Monte Carlo integration and named it Volumetric Obscurance [S2, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
[S2] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
