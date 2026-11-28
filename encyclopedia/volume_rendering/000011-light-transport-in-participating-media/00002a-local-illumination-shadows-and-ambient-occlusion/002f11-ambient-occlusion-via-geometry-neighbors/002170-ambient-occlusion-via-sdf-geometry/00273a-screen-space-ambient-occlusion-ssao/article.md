# Screen-Space Ambient Occlusion (SSAO)
Also called: Crytek's Screen-Space AO, Image Effect SSAO, PostProcess::SSAO, SSAO (Original), Screen-Space Ambient Occlusion (SSAO) Pass

## Overview
Screen-Space Ambient Occlusion (SSAO) is a post-processing technique that estimates ambient occlusion by sampling points in a hemisphere around each pixel's surface position and checking if those points are occluded by geometry in the depth buffer [S1, S2, S3, unofficial]. It solves the problem of real-time ambient occlusion by approximating the occlusion integral using only 2D screen-space data (depth and normal buffers) rather than querying the full 3D scene geometry [S2, S3, unofficial]. Unlike [SDF-based Ambient Occlusion](1881-sdf-based-ambient-occlusion.md) which queries a signed distance field, SSAO relies on the rendered image's depth buffer, making it significantly faster but limited to visible surfaces and prone to edge artifacts [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `hitPosTex`: Texture containing the world-space position of the fragment (32-bit float RGB) [S1, unofficial].
    *   `hitNormTex`: Texture containing the surface normal of the fragment (32-bit float RGB) [S1, unofficial].
    *   `depthTex`: Texture containing the depth value and cell size information (32-bit float) [S1, unofficial].
    *   `noiseTex`: Texture containing random vectors for tangent space generation (32-bit float RGB) [S1, unofficial].
    *   `projMat`: Projection matrix [S1, unofficial].
    *   `viewMat`: View matrix [S1, unofficial].
    *   `camPos`: Camera position in world space [S1, unofficial].
    *   `radius`: Sampling radius [S1, unofficial].
    *   `NUM_SAMPLES`: Number of samples per pixel [S1, unofficial].
*   **Output:**
    *   `outVisibility`: Float texture storing the final visibility factor (0.0 to 1.0) at location 0 [S1, unofficial].

**Data Structures:**
*   **Sample Kernel:** A pre-computed array of `NUM_SAMPLES` 3D vectors (`samples[i]`) distributed in a hemisphere or sphere [S1, unofficial].
*   **TBN Matrix:** A 3x3 matrix constructed from the tangent, bitangent, and normal vectors to rotate sample vectors into world space [S1, unofficial].

**Parameters:**
*   `radius`: The radius of the hemisphere used for sampling, determining the scale of occlusion [S1, unofficial]. Clamped between 0.05f and 1.0f [S2, unofficial].
*   `NUM_SAMPLES`: The number of random samples taken per pixel for the occlusion estimation [S1, unofficial].
*   `bias`: A small offset applied to the hit position to prevent self-intersection artifacts (acne) [S1, Parent, unofficial]. Derived from depth cell size, e.g., `hitDepth_cellSize.y / 5.0` [S1, unofficial].
*   `m_MinZ`: Minimum depth threshold to prevent self-occlusion artifacts near the camera [S2, unofficial]. Clamped between 0.00001f and 0.5f [S2, unofficial].
*   `m_OcclusionIntensity`: Multiplier for the final occlusion value [S2, unofficial]. Clamped between 0.5f and 4.0f [S2, unofficial].
*   `m_OcclusionAttenuation`: Controls the falloff of occlusion [S2, unofficial]. Clamped between 0.2f and 2.0f [S2, unofficial].
*   `m_Blur`: Number of blur passes (horizontal and vertical) [S2, unofficial]. Clamped between 0 and 4 [S2, unofficial]. If > 0, a separable blur is applied [S2, unofficial].
*   `m_Downsampling`: Integer factor to reduce the resolution of the SSAO calculation [S2, unofficial]. Clamped between 1 and 6 [S2, unofficial].

**Steps:**

1.  **Fetch Data:** For each pixel coordinate `coord`, fetch the hit position, normal, and depth from the input textures [S1, unofficial].
    $$ \mathbf{p} = \text{texelFetch}(\text{hitPosTex}, \text{coord}, 0).\text{xyz} $$
    $$ \mathbf{n} = \text{texelFetch}(\text{hitNormTex}, \text{coord}, 0).\text{xyz} $$
    $$ d = \text{texelFetch}(\text{depthTex}, \text{coord}, 0).\text{xyz} $$
    Discard if $\mathbf{p} = \mathbf{0}$ (background) [S1, unofficial].

2.  **Bias Position:** Offset the hit position along the normal to avoid self-intersection [S1, unofficial].
    $$ \mathbf{p}_{\text{biased}} = \mathbf{p} + \mathbf{n} \cdot \text{bias} $$
    The bias is often derived from the depth cell size, e.g., $\text{depth\_cellSize.y} / 5.0$ [S1, unofficial].

3.  **Construct TBN Matrix:** Generate a tangent vector $\mathbf{t}$ and bitangent $\mathbf{b}$ to form a basis for rotating samples [S1, unofficial].
    $$ \mathbf{t} = \text{normalize}(\mathbf{r} - \mathbf{n} \cdot (\mathbf{r} \cdot \mathbf{n})) $$
    $$ \mathbf{b} = \text{cross}(\mathbf{n}, \mathbf{t}) $$
    $$ \text{TBN} = [\mathbf{t}, \mathbf{b}, \mathbf{n}] $$
    where $\mathbf{r}$ is a random vector fetched from `noiseTex` [S1, unofficial].

4.  **Sample and Project:** For each sample $i$ from $0$ to $\text{NUM\_SAMPLES}-1$:
    a. Transform the sample vector $\mathbf{s}_i$ from tangent space to world space and scale by radius:
    $$ \mathbf{s}_{\text{world}} = \text{TBN} \cdot \mathbf{s}_i \cdot \text{radius} $$
    b. Compute the sample position in world space:
    $$ \mathbf{p}_{\text{samp}} = \mathbf{p}_{\text{biased}} + \mathbf{s}_{\text{world}} $$
    c. Project $\mathbf{p}_{\text{samp}}$ into screen space:
    $$ \mathbf{v}_{\text{clip}} = \text{projMat} \cdot \text{viewMat} \cdot [\mathbf{p}_{\text{samp}}, 1]^T $$
    $$ \mathbf{v}_{\text{ndc}} = \mathbf{v}_{\text{clip}}.\text{xyz} / \mathbf{v}_{\text{clip}}.\text{w} $$
    $$ \mathbf{v}_{\text{screen}} = \mathbf{v}_{\text{ndc}} \cdot 0.5 + 0.5 $$

5.  **Occlusion Check:**
    a. Check if $\mathbf{v}_{\text{screen}}$ is within screen bounds $[0, 1]$ [S1, unofficial].
    b. Fetch the depth $d_{\text{samp}}$ at $\mathbf{v}_{\text{screen}}$ from `depthTex`.
    c. Perform range check: If the distance between the sample depth and the original depth exceeds the radius, the sample is too far and contributes 0 [S1, unofficial].
    $$ \text{rangeCheck} = (|d - d_{\text{samp}}| < \text{radius}) ? 1.0 : 0.0 $$
    d. Check if the sample is occluded: If the sample depth $d_{\text{samp}}$ is less than the distance from the camera to the sample position, it is occluded [S1, unofficial].
    $$ \text{occlusion} += (\text{sampleDepth} < \text{length}(\mathbf{c}_{\text{pos}} - \mathbf{p}_{\text{samp}}) ? 1.0 : 0.0) \cdot \text{rangeCheck} $$

6.  **Normalize:**
    $$ \text{visibility} = 1.0 - \frac{\text{occlusion}}{\text{count}} $$
    If no samples were valid, visibility is 1.0 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ssao(coord, hitPosTex, hitNormTex, depthTex, noiseTex, projMat, viewMat, radius, num_samples, samples, camPos):
    # 1. Fetch Data
    hitPos = hitPosTex.fetch(coord)
    hitNorm = hitNormTex.fetch(coord)
    depthInfo = depthTex.fetch(coord)
    hitDepth = depthInfo.depth
    depthCellSize = depthInfo.cellSize
    
    # Discard background
    if hitPos == vec3(0, 0, 0):
        return 1.0

    # 2. Bias Position
    bias = depthCellSize.y / 5.0
    hitPosBiased = hitPos + hitNorm * bias

    # 3. Construct TBN
    noiseVec = noiseTex.fetch(coord)
    tangent = normalize(noiseVec - hitNorm * dot(noiseVec, hitNorm))
    bitangent = cross(hitNorm, tangent)
    tbn = mat3(tangent, bitangent, hitNorm)

    occlusion = 0.0
    count = 0

    # 4. Sample and Project
    for i in range(num_samples):
        # Transform sample to world space
        samp = tbn * samples[i]
        samp = hitPosBiased + samp * radius

        # Project to screen space
        offset = vec4(samp, 1.0)
        offset = projMat * viewMat * offset
        offset.xyz /= offset.w
        offset.xyz = offset.xyz * 0.5 + 0.5

        # Check bounds
        if all(lessThanEqual(offset.xy, vec2(1, 1))) and all(greaterThanEqual(offset.xy, vec2(0, 0))):
            # Fetch sample depth
            sampleDepth = depthTex.fetch(offset.xy).depth
            
            # Range check
            rangeCheck = 1.0 if abs(hitDepth - sampleDepth) < radius else 0.0
            
            # Occlusion check
            isOccluded = sampleDepth < length(camPos - samp)
            
            if isOccluded:
                occlusion += rangeCheck
            count += 1

    # 6. Normalize
    if count > 0:
        visibility = 1.0 - (occlusion / count)
    else:
        visibility = 1.0
        
    return visibility
```

## Subtopics

### [Combined Volumetric and Geometric Ambient Occlusion](14367-combined-volumetric-and-geometric-ambient-occlusion.md)
Computes mutual occlusion between scalar volumes and geometric primitives using screen-space vicinity terms for high-frequency details. Choose when rendering mixed media (volumes + streamlines) and depth perception is critical.

### [Screen-Space Ambient Occlusion for Volumes](16107-screen-space-ambient-occlusion-for-volumes.md)
Extends SSAO to volumes by extracting position/normal from raycasting to create a synthetic isocontour layer. Choose when applying SSAO to internal volume structures rather than just surface meshes.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples [S3, unofficial].
*   **Memory Complexity:** $O(N)$ for the input/output buffers (depth, normal, position textures) [S1, S2, unofficial]. Sample kernel storage complexity is not given in the sources.
*   **Strengths:** Real-time performance; works on any rendered geometry without scene traversal; simple to implement as a post-process [S3, unofficial].
*   **Limitations:** Limited to visible surfaces (no occlusion from back-facing or occluded geometry); suffers from "halo effect" (blackened edges) if not cosine-weighted [S3, unofficial]; artifacts at screen edges and depth discontinuities [S3, unofficial].
*   **When to use:** Real-time applications (games, interactive medical visualization) where performance is paramount and full geometric AO is too slow [S5, unofficial].
*   **When not to use:** Scenes requiring occlusion from non-visible geometry; when halo artifacts are unacceptable without additional filtering [S3, S5, unofficial].

## Background
SSAO was introduced by Crytek as the first screen-space method for ambient occlusion, using only the Z-Buffer to compute AO at full resolution [S3, unofficial]. It was developed to provide real-time ambient occlusion for games, balancing quality and performance [S3, unofficial]. The technique has been implemented in various libraries, including VTK (introduced in VTK 9.0 in July 2020 for surfaces, extended to volumes by 2023) [S5, unofficial]. Loos and Sloan later expanded on the approach, formulating it as Monte Carlo integration under the name "Volumetric Obscurance" [S3, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
[S2] dario-zubovic/metaballs, https://github.com/dario-zubovic/metaballs, fetched 2026-10-26 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] jose-villegas/StyleTransferFunction, https://github.com/jose-villegas/StyleTransferFunction, fetched 2026-10-27 (unofficial)
[S5] Enhancing Medical Imaging Technique: Ambient Occlusion ..., https://graylight-imaging.com/blog/enhancing-medical-imaging-technique-ambient-occlusion-in-volumetric-rendering/, fetched 2026-10-28 (unofficial)
