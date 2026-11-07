# Feature-Driven Ambient Occlusion

Also called: Feature-Driven AO

## Overview
Feature-Driven Ambient Occlusion is a shading technique for direct volume rendering (DVR) that computes ambient occlusion separately for identified features (layers) to prevent the overdarkening of internal layers caused by inter-occlusions from neighboring features, while preserving auto-occlusions [S1]. Unlike classical ambient occlusion, which darkens internal layers in multi-layer volumes due to inter-occlusions, this method neglects inter-occlusions between features to prevent overdarkening while keeping auto-occlusions [S1]. It builds on [Local Ambient Occlusion](4977-local-ambient-occlusion.md) by introducing feature-based isolation of occlusion calculations.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$, a transfer function mapping scalar values to opacity $\alpha$ and color, a set of identified features $k \in [1, n_f]$ where $n_f$ is the number of features, and parameters $L$ (number of rays), $M$ (samples per ray), $R_\Omega$ (occlusion sphere radius), and $a$ (initial offset) [S1].
*   **Output:** A final 3D ambient occlusion volume $AO(v)$ and rendered image [S1].

**Data Structures:**
*   **Bricks:** The 3D dataset is decomposed into equal-sized bricks. Each brick stores a boolean indicating if it contains only empty voxels. Bricks overlap to prevent discontinuities at borders [S1].
*   **Occlusion Textures:** $n_f$ separate 3D textures $T_k$, one for each feature $k$, storing the occlusion factor for voxels belonging to that feature [S1].

**Parameters:**
*   $L$: Number of cast rays used to sample the hemisphere for ambient occlusion computation [S1].
*   $M$: Number of samples used to approximate the integral for light contribution along a single ray [S1].
*   $R_\Omega$: Sphere radius for the occlusion region. If too large, it causes overocclusion; if too small, it results in poor shading [S1].
*   $a$: Initial offset to lower the influence of nearest neighbors [S1].
*   $n_f$: Number of features identified in the dataset [S1].

**Algorithm Steps:**

1.  **Preprocessing (Bricking):** Decompose the 3D dataset into equal-sized bricks. Scan each brick to determine if it contains only empty voxels, storing this as a boolean. Use overlapping bricks to prevent discontinuities at borders [S1].

2.  **Per-Feature Occlusion Computation:** For each feature $k \in [1, n_f]$:
    a.  Initialize a 3D texture $T_k$ for the occlusion factor.
    b.  For each voxel $v$ in the volume:
        i.  If $v$ does not belong to feature $k$, set $T_k(v) = 1.0$ (fully lit/no occlusion) or skip computation if the voxel is empty [S1].
        ii. If $v$ belongs to feature $k$, compute the ambient occlusion factor:
            1.  Cast $L$ rays uniformly scattered on the hemisphere centered at $v$'s normal (approximated from volume gradient) [S1].
            2.  For each ray $l$, compute the light contribution $I_l(x_v, k)$ using $M$ samples in a front-to-back compositing scheme:
                $$ I_l(x_v, k) = \frac{1}{M} \sum_{m=0}^{M-1} \prod_{i=0}^{m-1} (1 - \alpha_{i,k}) $$
                where $\alpha_{i,k}$ is the opacity of sample $i$ along ray $l$, obtained via the transfer function. Crucially, for all neighboring voxels (inside sphere radius $R_\Omega$) that do *not* belong to feature $k$, set their opacity to 0 during this calculation [S1].
            3.  Combine ray contributions to get the occlusion factor for voxel $v$ in texture $T_k$:
                $$ T_k(v) = \frac{1}{L} \sum_{l=1}^{L} I_l(x_v, k) $$
    c.  Optimize by raycasting only non-empty voxels (those not marked as empty in the brick structure) [S1].

3.  **Recombination:** Recombine the separate occlusion textures $T_k$ into the final ambient occlusion volume $AO(v)$ using the maximum operator:
    $$ AO(v) = \max \{ T_k(v) \mid \forall k \in [1, n_f] \} $$
    This operator is preferred over averaging because it reduces artifacts, improves contrast, and extends the influence of opaque features to neighboring less opaque ones, which is acceptable as less opaque features are less visible [S1].

4.  **Rendering:** Perform final rendering using a traditional volume raycasting algorithm (e.g., in GLSL). Fetch one additional texture for the precomputed ambient occlusion factor $AO(v)$ and two additional textures for preintegrated volume rendering [S1]. The light contribution $I_l(x_v)$ for a ray is approximated using the final AO factor:
    $$ I_l(x_v) = \sum_{m=0}^{M} \frac{1}{M} \prod_{i=0}^{m-1} (1 - \alpha_i) \cdot AO(x_v) $$
    (Note: The source formula [S1] Eq. 3 and 5 suggests the AO factor modulates the transmittance or is applied during compositing. The exact integration in the final pass uses the precomputed AO value) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feature_driven_ao(volume, features, transfer_function, L, M, R_Omega, a):
    """
    volume: 3D array of scalar values
    features: 3D array of feature IDs (0 for empty/background)
    transfer_function: function mapping scalar value to (opacity, color)
    L: number of rays
    M: number of samples per ray
    R_Omega: occlusion sphere radius
    a: initial offset
    """
    n_f = max(features.flatten()) + 1
    occlusion_textures = []
    
    # Preprocessing: Bricking
    bricks = decompose_into_bricks(volume)
    empty_bricks = scan_bricks_for_empty(bricks) # Boolean array
    
    for k in range(1, n_f + 1):
        T_k = create_3d_texture(volume.shape)
        
        for voxel in volume.voxels:
            if features[voxel] != k:
                T_k[voxel] = 1.0 # No occlusion for non-feature voxels in this texture
                continue
            
            # Check if voxel is in an empty brick (optimization)
            if is_in_empty_bricks(voxel, empty_bricks):
                T_k[voxel] = 1.0
                continue
                
            # Compute AO for feature k
            ao_sum = 0.0
            for l in range(L):
                ray = generate_ray(voxel, L) # Uniformly scattered on hemisphere
                light_contrib = 0.0
                
                # Front-to-back compositing
                transmittance = 1.0
                for m in range(M):
                    sample_pos = ray.origin + ray.direction * (a + m * (R_Omega - a) / M)
                    sample_val = interpolate_volume(volume, sample_pos)
                    
                    # Get opacity for feature k only
                    # If sample is not in feature k, opacity is 0
                    sample_feature = get_feature_at(sample_pos, features)
                    if sample_feature == k:
                        alpha = transfer_function(sample_val).opacity
                    else:
                        alpha = 0.0
                    
                    light_contrib += transmittance * alpha
                    transmittance *= (1 - alpha)
                
                light_contrib /= M
                ao_sum += light_contrib
            
            T_k[voxel] = ao_sum / L
        
        occlusion_textures.append(T_k)
    
    # Recombination
    AO_volume = create_3d_texture(volume.shape)
    for voxel in volume.voxels:
        max_ao = 0.0
        for T_k in occlusion_textures:
            if T_k[voxel] > max_ao:
                max_ao = T_k[voxel]
        AO_volume[voxel] = max_ao
        
    return AO_volume
```

**Edge Cases and Pitfalls:**
*   **Visual Artifacts:** Naively setting opacity of neighboring features to 0 causes visual artifacts (white dots) at feature borders due to ray sampling and trilinear interpolation when there is a connected area with a different ambient occlusion value [S1].
*   **Inaccurate Recombination:** Averaging all occlusion textures $T_k$ results in inaccurate ambient occlusion values [S1].
*   **Parameter Sensitivity:** The sphere radius $R_\Omega$ is critical. If too large, it causes overocclusion; if too small, it results in poor shading [S1].

## Operating characteristics
*   **Time Complexity:** Computing separate ambient occlusion textures increases computation cost by a factor equal to the number of features $n_f$ [S1].
*   **Memory Complexity:** Requires storing $n_f$ separate 3D occlusion textures plus the brick structure [S1].
*   **Measured Performance:**
    *   Dataset: Orange (256x256x64), AMD Opteron 2431, Nvidia GTX 295, 1024x768 viewport: No shading ~200 fps, Phong shading ~125 fps, Classical AO ~130 fps, Feature-driven AO ~130 fps [S1].
    *   Dataset: CT Knee (379x229x305), AMD Opteron 2431, Nvidia GTX 295, 1024x768 viewport: No shading ~60 fps, Phong shading ~50 fps, Classical AO ~45 fps, Feature-driven AO ~45 fps [S1].
*   **Strengths:** Improves visual quality of images compared to classical ambient occlusion, less parameter-sensitive, improves usability for non-experts, preserves auto-occlusions [S1].
*   **Limitations:** Increased preprocessing time and memory consumption due to bricking and multiple occlusion textures [S1]. Requires pre-segmented features [S1].
*   **When to use:** Direct volume rendering of multi-layer datasets, such as CT scans, where features are interleaved and overdarkening is a concern [S1].
*   **When not to use:** When features are not clearly defined or pre-segmented, or when computational resources are severely limited and preprocessing time is prohibitive [S1].

## Background
Feature-Driven Ambient Occlusion was introduced by A. Ancel, J.-M. Dischler, and C. Mongenet in 2010 [S1]. It addresses the limitations of straightforward application of ambient occlusion in direct volume rendering, which results in overdarkening of internal layers in multi-layer volumes [S1]. The method uses a GPU-based approach with bricking to speed up computations [S1]. It is used in direct volume rendering (DVR) of multi-layer datasets, such as CT scans, to improve visual quality and usability for non-experts [S1]. Unlike Correa and Ma [CM09] who use occlusion for data classification, this method uses feature classification to improve shading [S1].

## Sources
[S1] Feature-Driven Ambient Occlusion for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/085-092, fetched 2026-10-26
