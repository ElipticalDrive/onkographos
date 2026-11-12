# 3D Gaussian Splatting Rendering with Language Features

Also called: LangSplat Rendering, Language Feature Rasterization

## Overview
This method extends the standard 3D Gaussian Splatting rasterizer to simultaneously render a language feature image alongside the RGB image by passing precomputed per-Gaussian language features to the rasterization backend [S1, unofficial]. It differs from standard 3DGS rendering by adding a parallel feature channel output and normalizing features per Gaussian before rasterization to prevent magnitude bias [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `pc`: A Gaussian primitive container object holding 3D Gaussians.
    *   `opt`: Configuration object containing `include_feature` (bool).
    *   `pipe`: Pipeline configuration object containing `compute_cov3D_python` (bool), `convert_SHs_python` (bool), and `scaling_modifier` (float).
    *   `bg_color`: Background color tensor of shape `[3]` on GPU.
*   **Outputs:**
    *   `rendered_image`: RGB image tensor.
    *   `language_feature_image`: Language feature image tensor.
    *   `radii`: Screen-space radii tensor.
    *   Dictionary containing `"render"`, `"language_feature_image"`, `"viewspace_points"`, `"visibility_filter"`, and `"radii"`.

**Data Structures:**
*   **Per-Gaussian Language Features:** A tensor retrieved from `pc`. The dimensionality depends on the feature level (e.g., 3D or higher), but the rasterization backend expects a feature tensor per Gaussian.
*   **GaussianRasterizationSettings:** Configuration passed to the CUDA rasterizer, specifically setting `include_feature=opt.include_feature`.

**Steps:**
1.  Check the `opt.include_feature` flag.
2.  If `opt.include_feature` is true:
    a. Retrieve per-Gaussian language features via `pc.get_language_feature()`.
    b. L2-normalize the language features per Gaussian to prevent magnitude bias during blending. The normalization formula is:
       $$
       \mathbf{f}_{norm} = \frac{\mathbf{f}}{\|\mathbf{f}\|_2 + \epsilon}
       $$
       where $\mathbf{f}$ is the feature vector, $\|\mathbf{f}\|_2$ is the L2 norm along the last dimension, and $\epsilon = 10^{-9}$ [S1, unofficial].
    c. Set `language_feature_precomp` to the normalized features.
3.  If `opt.include_feature` is false:
    a. Set `language_feature_precomp` to a dummy tensor of shape `(1,)` matching the dtype and device of `opacity` [S1, unofficial].
4.  Prepare covariance and color data based on `pipe` settings:
    a. If `pipe.compute_cov3D_python` is true, compute 3D covariance in Python via `pc.get_covariance(scaling_modifier)`; otherwise, pass `scales` and `rotations` to the rasterizer [S1, unofficial].
    b. If `pipe.convert_SHs_python` is true, evaluate spherical harmonics in Python using `eval_sh`; otherwise, pass raw SH coefficients (`shs`) or precomputed colors (`colors_precomp`) [S1, unofficial].
5.  Configure `GaussianRasterizationSettings` with `include_feature=opt.include_feature` [S1, unofficial].
6.  Call the `rasterizer` with the following arguments:
    *   `means3D`: 3D positions.
    *   `means2D`: 2D projected positions.
    *   `shs` / `colors_precomp`: Spherical harmonics coefficients or precomputed colors.
    *   `language_feature_precomp`: The prepared language features.
    *   `opacity`: Opacity values.
    *   `scales` / `rotations` / `cov3D_precomp`: Scale, rotation, or precomputed covariance matrices.
7.  The rasterizer returns `rendered_image`, `language_feature_image`, and `radii` [S1, unofficial].
8.  Construct and return a dictionary with keys:
    *   `"render"`: `rendered_image`.
    *   `"language_feature_image"`: `language_feature_image`.
    *   `"viewspace_points"`: Gradients for 2D means (computed by the rasterizer).
    *   `"visibility_filter"`: Boolean mask where `radii > 0`.
    *   `"radii"`: Screen-space radii [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_language_features(pc, opt, pipe, bg_color):
    # Step 1-3: Prepare language features
    if opt.include_feature:
        language_feature_precomp = pc.get_language_feature()
        # L2-normalize to prevent magnitude bias
        # Add epsilon to avoid division by zero
        language_feature_precomp = language_feature_precomp / (
            language_feature_precomp.norm(dim=-1, keepdim=True) + 1e-9
        )
    else:
        # Dummy tensor matching dtype and device of opacity
        language_feature_precomp = torch.zeros(1, dtype=pc.opacity.dtype, device=pc.opacity.device)

    # Step 4: Prepare covariance and colors
    if pipe.compute_cov3D_python:
        cov3D_precomp = pc.get_covariance(pipe.scaling_modifier)
        scales = None
        rotations = None
    else:
        cov3D_precomp = None
        scales = pc.scales
        rotations = pc.rotations

    if pipe.convert_SHs_python:
        # Evaluate SH in Python; assume eval_sh is defined elsewhere or handled by pipe
        # For the rasterizer call, we typically pass precomputed colors if SH conversion is done
        # However, the source code snippet implies passing shs/colors_precomp directly to rasterizer
        # based on pipe flags. If convert_SHs_python is True, we might pass colors_precomp
        # or the rasterizer handles it. The source snippet shows passing shs/colors_precomp.
        # We assume colors_precomp is precomputed if convert_SHs_python is True, 
        # or shs is passed if False.
        colors_precomp = pc.get_sh_colors() # Placeholder for SH evaluation logic
        shs = None
    else:
        colors_precomp = None
        shs = pc.shs

    # Step 5: Configure rasterization settings
    rasterization_settings = GaussianRasterizationSettings(
        image_height=..., # Not given in sources, assumed from context
        image_width=..., # Not given in sources, assumed from context
        tanfovx=...,     # Not given in sources, assumed from context
        tanfovy=...,     # Not given in sources, assumed from context
        bg=bg_color,
        scale_modifier=pipe.scaling_modifier,
        viewmatrix=..., # Not given in sources, assumed from context
        projmatrix=..., # Not given in sources, assumed from context
        sh_degree=...,  # Not given in sources, assumed from context
        include_feature=opt.include_feature
    )

    # Step 6: Call rasterizer
    rasterizer = GaussianRasterizer(rasterization_settings=rasterization_settings)
    
    rendered_image, language_feature_image, radii = rasterizer(
        means3D=pc.means3D,
        means2D=pc.means2D,
        shs=shs,
        colors_precomp=colors_precomp,
        language_feature_precomp=language_feature_precomp,
        opacities=pc.opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp
    )

    # Step 7-8: Return results
    visibility_filter = radii > 0
    return {
        "render": rendered_image,
        "language_feature_image": language_feature_image,
        "viewspace_points": pc.means2D, # Gradients for 2D means
        "visibility_filter": visibility_filter,
        "radii": radii
    }
```

**Parameters:**
*   `opt.include_feature` (bool): Flag to enable/disable language feature rendering and computation [S1, unofficial].
*   `scaling_modifier` (float, default 1.0): Multiplier applied to Gaussian scales during rasterization [S1, unofficial].
*   `pipe.compute_cov3D_python` (bool): If true, computes 3D covariance in Python; otherwise passes `scales` and `rotations` [S1, unofficial].
*   `pipe.convert_SHs_python` (bool): If true, evaluates spherical harmonics in Python; otherwise passes raw SH coefficients [S1, unofficial].
*   `bg_color` (torch.Tensor, shape [3]): Background color tensor on GPU, set to `[1,1,1]` for white background or `[0,0,0]` for black [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Magnitude Bias:** Failure to L2-normalize language features can lead to magnitude bias during blending, causing incorrect feature interpolation [S1, unofficial].
*   **Feature Dimensionality:** The decoder elevates features from 3 dimensions to 512 dimensions for further operations, implying the rasterized language feature image may be low-dimensional (e.g., 3D) and requires post-processing for high-dimensional tasks [S1, unofficial].
*   **Dummy Tensor:** When features are disabled, a dummy tensor of shape `(1,)` must be passed to maintain tensor shape consistency in the rasterizer call [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables simultaneous rendering of semantic/language information alongside geometry and appearance, facilitating tasks like 3D object localization and semantic segmentation [S1, unofficial].
*   **Limitations:** Requires precomputed language features and an additional normalization step. The output language feature image is low-dimensional (3D) and requires a decoder for high-dimensional applications [S1, unofficial].
*   **When to use:** When semantic or language-aware rendering is required in addition to photorealistic rendering.
*   **When not to use:** When only photorealistic rendering is needed, as it adds computational overhead for feature normalization and parallel channel rendering.

## Background
*   **Origin:** LangSplat was introduced by Minghan Qin, Wanhua Li, Jiawei Zhou, Haoqian Wang, and Hanspeter Pfister in the paper "LangSplat: 3D Language Gaussian Splatting" (CVPR 2024) [S1, unofficial].
*   **History:** The method extends the original 3D Gaussian Splatting framework to include language features, allowing for 3D language understanding and interaction. It builds upon the differentiable rasterization of 3D Gaussians.
*   **Usage:** Used for 3D object localization on LERF and 3D semantic segmentation on LERF, leveraging preprocessed datasets like 3D-OVS with language features [S1, unofficial].

## Sources
[S1] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
