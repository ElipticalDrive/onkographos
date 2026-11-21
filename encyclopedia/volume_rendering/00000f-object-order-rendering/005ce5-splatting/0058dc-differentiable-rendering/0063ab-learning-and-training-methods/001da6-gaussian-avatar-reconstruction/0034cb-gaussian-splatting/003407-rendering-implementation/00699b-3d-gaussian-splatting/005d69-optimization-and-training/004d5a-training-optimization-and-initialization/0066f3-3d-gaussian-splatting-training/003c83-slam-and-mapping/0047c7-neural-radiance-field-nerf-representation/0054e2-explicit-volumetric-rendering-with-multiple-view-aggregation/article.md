# Explicit Volumetric Rendering with Multiple View Aggregation

Also called: Pooled Latent Volume Rendering, RGBα Volume Rendering

## Overview
This method models scene geometry and appearance as an explicit RGBα volume to amortize rendering across multiple views, solving the computational inefficiency of NeRF-like methods where per-ray sampling is independent and not reusable across different viewing angles [S1]. Unlike its siblings [Scene NeRF](12484-scene-nerf.md), [NeRF-OSR](14477-nerf-osr.md), and [Dynamic Neural Radiance Fields](15318-dynamic-neural-radiance-fields.md), which typically rely on implicit field evaluations or sparse ray sampling, this approach generates a dense volumetric representation that enables real-time rendering and the use of structural losses like SSIM during training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ source views $S = \{I_1, \dots, I_n\}$ with corresponding camera poses $P = \{P_1, \dots, P_n\}$ and intrinsic parameters.
*   **Outputs:** An RGBα volume $V \in \mathbb{R}^{4 \times d_s \times h \times w}$ and rendered target images $\hat{C}$.

**Data Structures:**
*   **Latent Volume ($Z_i$):** An intermediate volumetric representation for each source view $i$. The exact dimensions are not specified in the sources, but it is processed by a 3D U-Net [S1].
*   **RGBα Volume ($V$):** A 4D tensor with channels for Red, Green, Blue, and Alpha (transparency). It has dimensions $4 \times d_s \times h \times w$, where $d_s$ is the number of depth samples, and $h, w$ are the spatial dimensions of the viewing frustum [S1].

**Numbered Steps:**
1.  **Encode Source Views:** For each source view $i \in \{1, \dots, n\}$, encode the image $I_i$ into a latent volume $Z_i$ using the encoding process described in the parent article's Section 3.1 [S1].
2.  **Select Origin:** Select an arbitrary source view $i^*$ from the set of source views to serve as the origin of the coordinate system [S1].
3.  **Align Volumes:** Align each latent volume $Z_i$ to the origin $i^*$ using the relative transformation $P_{i, i^*}$ between the pose $P_i$ of view $i$ and the pose $P_{i^*}$ of the origin [S1]. Let $\mathcal{T}(P_{i, i^*}, Z_i)$ denote this aligned volume.
4.  **Pool Volumes:** Compute the pooled volumetric latent $\bar{Z}$ by taking the mean of the aligned latent volumes across all $n$ views:
    $$ \bar{\mathbf{Z}} = \frac{1}{n} \sum_{i \in n} \mathcal{T}(\mathbf{P}_{i, i^*}, \mathbf{Z}_i) $$
    [S1]
5.  **Generate RGBα Volume:** Feed the pooled latent $\bar{Z}$ into a 3D U-Net to generate the explicit RGBα volume $V$ [S1].
6.  **Perspective Deformation:** Apply perspective deformation to the viewing frustum using the intrinsic camera parameters. This is achieved via inverse warping and trilinear sampling [S1].
7.  **Render:** For a given pixel location $(i, j)$, calculate the expected color $\hat{C}_{i,j}$ by integrating along the depth samples $k=1 \dots d_s$:
    $$ \hat{C}_{i,j} = \sum_{k=1}^{d_s} T_{i,j}^k \alpha_{i,j}^k \mathbf{c}_{i,j}^k $$
    where the transmittance $T_{i,j}^k$ is defined as:
    $$ T_{i,j}^k = \prod_{m=1}^{k-1} (1-\alpha_{i,j}^m) $$
    Here, $\mathbf{c}_{i,j}^k$ is the color value from the first 3 channels of $V$ at sample $k$, and $\alpha_{i,j}^k$ is the alpha value from the last channel of $V$ at sample $k$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_explicit_volume(source_views, poses, intrinsics, origin_index):
    """
    source_views: List of images [I_1, ..., I_n]
    poses: List of camera poses [P_1, ..., P_n]
    intrinsics: Camera intrinsic parameters
    origin_index: Index i* of the reference view
    """
    n = len(source_views)
    aligned_latents = []
    
    # Step 1 & 2 & 3: Encode and Align
    for i in range(n):
        # Encode view i into latent volume Z_i
        # Note: Exact encoding architecture details are in parent article Sect 3.1 [S1]
        Z_i = encode_view(source_views[i])
        
        # Calculate relative transformation from view i to origin view
        P_i_star = poses[origin_index]
        P_i = poses[i]
        P_rel = compute_relative_transform(P_i, P_i_star)
        
        # Align Z_i to the origin coordinate system
        Z_aligned = transform_volume(Z_i, P_rel)
        aligned_latents.append(Z_aligned)
        
    # Step 4: Pool aligned volumes
    # Sum the aligned volumes and divide by n
    pooled_latent = aligned_latents[0]
    for k in range(1, n):
        pooled_latent = pooled_latent + aligned_latents[k]
    pooled_latent = pooled_latent / n
    
    # Step 5: Generate RGBα Volume
    # Feed pooled latent to 3D U-Net
    V = generate_rgb_alpha_volume(pooled_latent)
    
    # Step 6 & 7: Render
    # Apply perspective deformation and integrate
    rendered_image = perspective_deform_and_render(V, intrinsics)
    
    return rendered_image

def perspective_deform_and_render(V, intrinsics):
    """
    V: RGBα volume [4, ds, h, w]
    intrinsics: Camera intrinsic matrix K
    """
    # Inverse warping and trilinear sampling based on intrinsics [S1]
    # This step maps the volume V to the target image plane
    # The exact implementation of inverse warping is referenced in Appendix of [S1]
    # For implementation, one would typically:
    # 1. Define a grid of 3D points in the camera coordinate system
    # 2. Project them to 2D pixel coordinates using K
    # 3. Sample V at these 3D locations using trilinear interpolation
    # 4. Perform alpha compositing along the ray
    
    # Placeholder for the specific inverse warping logic detailed in [S1] Appendix
    # Since the exact warping code is not provided in the main text, 
    # we assume standard inverse warping using K [S1].
    
    # Rendered color calculation per pixel (i,j)
    # See formula in text for integration over ds samples
    pass
```

**Parameters:**
*   $\lambda$: Weight for the SSIM loss in the training objective. Typical value not given in the sources [S1].
*   $d_s$: Number of depth samples. Value not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Coordinate System Alignment:** The choice of origin view $i^*$ is arbitrary, but all latent volumes must be consistently aligned to this origin to ensure correct pooling [S1].
*   **Precision:** Trilinear sampling is used for perspective deformation; care must be taken with boundary conditions and interpolation artifacts [S1].
*   **Degenerate Input:** The method assumes a valid set of source views with known poses. If poses are inaccurate, the alignment and pooling steps will produce incorrect volumetric representations [S1].

## Subtopics
*   **Scene NeRF [Scene NeRF](12484-scene-nerf.md):** Focuses on large-scale scene reconstruction. Choose when scalability to large environments is required.
*   **NeRF-OSR [NeRF-OSR](14477-nerf-osr.md):** Handles out-of-sample rendering. Choose when generalizing to unseen objects is critical.
*   **Dynamic Neural Radiance Fields [Dynamic Neural Radiance Fields](15318-dynamic-neural-radiance-fields.md):** Models time-varying scenes. Choose when dealing with dynamic content.

## Operating characteristics
*   **Time Complexity:** Rendering is amortized across views. Per-view rendering time is 0.0178s (56 FPS) on an NVIDIA Tesla V100 [S1].
*   **Memory Complexity:** Requires storing the RGBα volume $V$ and latent volumes $Z_i$. Exact memory footprint not given in the sources [S1].
*   **Measured Performance:** On an NVIDIA Tesla V100, rendering 250 views takes 1.022s (245 FPS), which is a 400x speedup over pixelNeRF's 474.8606s (0.5 FPS) [S1]. Per-view, it is 100x faster than pixelNeRF's 1.9047s [S1].
*   **Strengths:** Real-time rendering capability; supports structural losses like SSIM during training because it renders full images rather than sparse rays [S1].
*   **Limitations:** Requires explicit volumetric storage; performance depends on the resolution of the RGBα volume [S1].
*   **When to Use:** When real-time novel view synthesis is required and multiple source views are available. Evaluated on ShapeNet chairs and cars [S1].
*   **When Not to Use:** When memory is extremely constrained and explicit volumes cannot be stored, or when only a single view is available for initialization.

## Background
The method was introduced in "Fast and Explicit Neural View Synthesis" [S1]. It addresses the limitation of NeRF-like methods where rendering cannot be amortized across views because each ray integral is independent [S1]. By using an explicit RGBα volume, the method allows for efficient rendering and the application of structural losses, which are not directly applicable to NeRF-like methods due to their sparse ray sampling [S1]. It has been evaluated on ShapeNet chairs and cars categories, demonstrating competitive visual quality and significant speed improvements over baselines like pixelNeRF [S1].

## Sources
[S1] Fast and Explicit Neural View Synthesis, https://doi.org/10.1109/wacv51458.2022.00009, fetched 2026-10-30
