# Feature-Driven Ambient Occlusion

Also called: Feature-Driven AO

## Overview
Feature-Driven Ambient Occlusion is a shading technique for direct volume rendering that computes ambient occlusion separately for identified features (layers) to prevent the overdarkening of internal layers caused by inter-occlusions from neighboring features, while preserving auto-occlusions [S1]. Unlike classical ambient occlusion which darkens internal layers in multi-layer volumes due to inter-occlusions, this method neglects inter-occlusions between features to prevent overdarkening while keeping auto-occlusions [S1]. It is distinct from [Local Ambient Occlusion](4977-local-ambient-occlusion.md) and [Local Ambient Occlusion (LAO)](4502-local-ambient-occlusion-lao.md) by explicitly segmenting the volume into features and computing independent occlusion volumes for each, rather than treating the volume as a single homogeneous medium or using statistical distributions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar dataset $V$, a set of identified features $F$ where $F(v) \in [1, n_f]$ and $n_f$ is the number of features, a transfer function mapping scalar values to opacity $\alpha$, the sphere radius $R_\Omega$, the number of rays $L$, and the number of samples per ray $M$ [S1].
*   **Output:** A final 3D ambient occlusion volume $AO$ [S1].

**Data Structures:**
*   **Bricks:** The 3D dataset is decomposed into equal-sized bricks [S1]. Each brick stores a boolean indicating if it contains only empty voxels [S1]. Bricks overlap to prevent discontinuities at borders [S1].
*   **Occlusion Textures:** A set of 3D textures $T_k$, one for each feature $k \in [1, n_f]$, storing the precomputed ambient occlusion factor for that feature [S1].
*   **Final AO Volume:** A 3D texture storing the recombined ambient occlusion values [S1].

**Parameters:**
*   $L$: Number of cast rays used to sample the sphere for ambient occlusion computation [S1].
*   $M$: Number of samples used to approximate the integral for light contribution along a single ray [S1].
*   $R_\Omega$: Sphere radius used for the occlusion region. If too large, it causes overocclusion; if too small, it results in poor shading [S1].
*   $n_f$: Number of features identified in the dataset [S1].
*   $\alpha_i$: Opacity of sample $i$, obtained via the transfer function [S1].

**Steps:**
1.  **Preprocessing (Bricking):** Decompose the 3D dataset into equal-sized bricks [S1]. Scan each brick to determine if it contains only empty voxels, storing this as a boolean [S1]. Use overlapping bricks to prevent discontinuities at borders [S1].
2.  **Per-Feature Occlusion Computation:** For each feature $k$ from $1$ to $n_f$: Initialize an empty 3D texture $T_k$ of the same dimensions as the dataset [S1]. For each voxel $v$ belonging to feature $k$, compute the ambient occlusion factor by casting $L$ rays [S1]. For each ray, compute the light contribution $I_l(v)$ using $M$ samples in a front-to-back compositing scheme [S1]. For every sample along the ray, if the sample belongs to feature $k$, use its actual opacity $\alpha_i$; if it does not belong to feature $k$, set its opacity to $0$ [S1]. Average the light contributions over $L$ rays to get $AO_k(v)$ [S1]. Store $AO_k(v)$ in $T_k(v)$ [S1].
3.  **Recombination:** For each voxel $v$, compute the final ambient occlusion value $AO(v)$ by taking the maximum of the per-feature occlusion values:
    $$AO(v) = \max \{T_k(v) \mid \forall k \in [1, n_f]\}$$
    This operator is preferred over averaging because it reduces artifacts, improves contrast, and extends the influence of opaque features to neighboring less opaque ones, which is acceptable as less opaque features are less visible [S1].
4.  **Rendering:** Perform final rendering using a traditional volume raycasting algorithm in GLSL [S1]. Fetch one additional texture for the precomputed ambient occlusion factor and two additional textures for preintegrated volume rendering [S1].

**Formulas:**
The light contribution $I_l(x_v)$ for a single ray is approximated using $M$ samples in a front-to-back compositing scheme:
$$I_l(x_v) = \frac{1}{M} \sum_{m=0}^{M-1} \prod_{i=0}^{m-1} (1 - \alpha_{i,k})$$
where $\alpha_{i,k}$ is the feature-dependent opacity of the $i$-th sample along the ray [S1]. The total light contribution for a voxel is the sum over all features:
$$I_l(x_v) = \sum_{k=1}^{n_f} I_l(x_v, k)$$
The final ambient occlusion factor for voxel $v$ is the average over $L$ rays:
$$AO(v) = \frac{1}{L} \sum_{l=1}^{L} I_l(x_v)$$
[Note: While the classic definition integrates over a hemisphere, volume rendering typically lacks surface normals; the source specifies using a sphere with rays uniformly scattered using polar coordinates [S1].]

**Pitfalls and Edge Cases:**
*   **Border Artifacts:** Naively setting the opacity of neighboring features to $0$ causes visual artifacts (white dots) at feature borders due to ray sampling and trilinear interpolation when there is a connected area with a different ambient occlusion value [S1].
*   **Recombination Choice:** Averaging all occlusion textures $T_k$ results in inaccurate ambient occlusion values [S1]. The maximum operator is required to maintain correct visibility cues [S1].
*   **Radius Sensitivity:** Choosing $R_\Omega$ too large causes overocclusion, while choosing it too small results in poor shading [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_feature_driven_ao(dataset, features, transfer_function, R_omega, L, M, brick_size):
    """
    dataset: 3D numpy array of scalars
    features: 3D numpy array of feature IDs (0 for empty/background)
    transfer_function: callable(scalar) -> float opacity in [0, 1]
    R_omega: float, occlusion sphere radius
    L: int, number of rays
    M: int, number of samples per ray
    brick_size: int, size of each brick dimension
    """
    n_f = int(features.max()) + 1
    volume_shape = dataset.shape
    H, W, D = volume_shape

    # Helper: Trilinear interpolation with boundary clamping
    def trilinear_interpolate(arr, x, y, z):
        x, y, z = np.clip([x, y, z], 0, [W-1, H-1, D-1])
        x0, y0, z0 = np.floor([x, y, z]).astype(int)
        x1, y1, z1 = np.clip(x0 + 1, 0, W-1), np.clip(y0 + 1, 0, H-1), np.clip(z0 + 1, 0, D-1)
        fx, fy, fz = x - x0, y - y0, z - z0
        v000 = arr[z0, y0, x0]; v100 = arr[z0, y0, x1]
        v010 = arr[z0, y1, x0]; v110 = arr[z0, y1, x1]
        v001 = arr[z1, y0, x0]; v101 = arr[z1, y0, x1]
        v011 = arr[z1, y1, x0]; v111 = arr[z1, y1, x1]
        c00 = v000*(1-fx) + v100*fx; c10 = v010*(1-fx) + v110*fx
        c01 = v001*(1-fx) + v101*fx; c11 = v011*(1-fx) + v111*fx
        c0 = c00*(1-fy) + c10*fy; c1 = c01*(1-fy) + c11*fy
        return c0*(1-fz) + c1*fz

    # Helper: Generate L rays uniformly on a sphere
    def generate_uniform_rays(num_rays):
        rays = []
        for _ in range(num_rays):
            theta = np.random.uniform(0, 2 * np.pi)
            phi = np.arccos(np.random.uniform(-1, 1))
            x = np.sin(phi) * np.cos(theta)
            y = np.sin(phi) * np.sin(theta)
            z = np.cos(phi)
            rays.append(np.array([x, y, z]))
        return rays

    # Preprocessing: Bricks
    # Decompose dataset into equal-sized bricks and scan for empty voxels per feature [S1]
    # Overlapping bricks are used to prevent discontinuities at borders [S1]
    brick_masks = {}
    for k in range(1, n_f):
        brick_masks[k] = np.zeros((H // brick_size + 1, W // brick_size + 1, D // brick_size + 1), dtype=bool)
        for bz in range(brick_masks[k].shape[0]):
            for by in range(brick_masks[k].shape[1]):
                for bx in range(brick_masks[k].shape[2]):
                    z_start, z_end = bz * brick_size, min((bz + 1) * brick_size, D)
                    y_start, y_end = by * brick_size, min((by + 1) * brick_size, H)
                    x_start, x_end = bx * brick_size, min((bx + 1) * brick_size, W)
                    brick_region = features[z_start:z_end, y_start:y_end, x_start:x_end]
                    brick_masks[k][bz, by, bx] = np.any(brick_region == k)

    rays = generate_uniform_rays(L)
    occlusion_textures = []

    # Per-Feature Occlusion Computation
    for k in range(1, n_f):
        T_k = np.zeros(volume_shape)
        for z in range(D):
            for y in range(H):
                for x in range(W):
                    if features[z, y, x] != k:
                        T_k[z, y, x] = 1.0
                        continue

                        # Optimization: skip bricks marked as empty for feature k [S1]
                    bz, by, bx = z // brick_size, y // brick_size, x // brick_size
                    if not brick_masks[k][bz, by, bx]:
                        T_k[z, y, x] = 1.0
                        continue

                    ao_sum = 0.0
                    for ray in rays:
                        light_contribution = 0.0
                        product_term = 1.0
                        for m in range(M):
                            # Sample position along ray within R_omega
                            sample_dist = (m + 0.5) * (R_omega / M)
                            sx, sy, sz = x + ray[0] * sample_dist, y + ray[1] * sample_dist, z + ray[2] * sample_dist
                            val = trilinear_interpolate(dataset, sx, sy, sz)
                            alpha = transfer_function(val)

                            # Ignore other features by setting opacity to 0 [S1]
                            feat_id = int(trilinear_interpolate(features, sx, sy, sz))
                            if feat_id != k:
                                alpha = 0.0

                            light_contribution += product_term
                            product_term *= (1.0 - alpha)
                        ao_sum += light_contribution / M
                    T_k[z, y, x] = ao_sum / L
        occlusion_textures.append(T_k)

    # Recombination
    final_ao = np.zeros(volume_shape)
    for z in range(D):
        for y in range(H):
            for x in range(W):
                max_val = 0.0
                for T_k in occlusion_textures:
                    val = T_k[z, y, x]
                    if val > max_val:
                        max_val = val
                final_ao[z, y, x] = max_val
    return final_ao
```

## Operating characteristics
*   **Time Complexity:** Computing separate ambient occlusion textures increases computation cost by a factor equal to the number of features $n_f$ [S1].
*   **Memory Complexity:** Requires storing $n_f$ occlusion 3D textures plus the final recombined texture, plus brick metadata [S1].
*   **Measured Performance:**
    *   Dataset: Orange (256x256x64), Viewport: 1024x768, Hardware: AMD Opteron 2431, Nvidia GTX 295.
        *   No shading: ~200 fps
        *   Phong shading: ~125 fps
        *   Classical AO: ~130 fps
        *   Feature-driven AO: ~130 fps [S1]
    *   Dataset: CT Knee (379x229x305), Viewport: 1024x768, Hardware: AMD Opteron 2431, Nvidia GTX 295.
        *   No shading: ~60 fps
        *   Phong shading: ~50 fps
        *   Classical AO: ~45 fps
        *   Feature-driven AO: ~45 fps [S1]
*   **Strengths:** Improves visual quality of multi-layer volumes by preventing overdarkening of internal layers; less parameter-sensitive than classical AO; preserves auto-occlusions for shape perception [S1].
*   **Limitations:** Requires pre-segmentation of features; computation cost scales linearly with the number of features; naive implementation causes artifacts at feature borders if not handled with separate textures and max recombination [S1].
*   **When to use:** Direct volume rendering of multi-layer datasets (e.g., CT scans) where features are interleaved and distinct visual separation is desired [S1].
*   **When not to use:** Single-feature volumes (classical AO is sufficient); scenes where inter-feature occlusion is desired (e.g., realistic shadowing between distinct objects).

## Background
Feature-Driven Ambient Occlusion was introduced by Ancel, Dischler, and Mongenet in 2010 [S1]. It addresses the limitations of straightforward ambient occlusion application in direct volume rendering, specifically the overdarkening of internal layers in multi-layer volumes [S1]. The method builds on [Local Ambient Occlusion](4977-local-ambient-occlusion.md) principles but introduces feature segmentation to decouple inter-occlusions [S1]. It is used in scientific 3D scalar field visualization to improve feature perception and data comprehension for non-experts [S1].

## Sources
[S1] Feature-Driven Ambient Occlusion for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/085-092, fetched 2026-10-26
