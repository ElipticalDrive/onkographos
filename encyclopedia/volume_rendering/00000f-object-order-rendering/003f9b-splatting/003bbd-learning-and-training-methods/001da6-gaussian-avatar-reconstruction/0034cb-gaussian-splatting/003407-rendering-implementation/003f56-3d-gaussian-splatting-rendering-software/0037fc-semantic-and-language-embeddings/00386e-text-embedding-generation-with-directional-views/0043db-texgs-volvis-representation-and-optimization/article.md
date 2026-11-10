# TexGS-VolVis Representation and Optimization

Also called: Textured Gaussian Splatting for Volume Visualization, TexGS

## Overview
TexGS-VolVis is a method for representing and editing volume visualization (VolVis) scenes using textured 2D Gaussian Splatting (2DGS) [S1]. It solves the problem of decoupling geometry from appearance in text-driven non-photorealistic style editing (NPSE), allowing for geometry-consistent stylization without the elliptical artifacts common in coupled representations [S1]. Unlike methods that freeze geometry to prevent distortion, TexGS-VolVis uses explicit 2D texture maps to store appearance, preserving geometric fidelity while enabling rich visual expressiveness [S1].

## How it works
**Inputs:** Multi-view images of a volumetric dataset; text prompts for style transfer or refinement.
**Outputs:** A scene representation consisting of 2D Gaussian primitives with geometry, shading, and texture attributes; rendered stylized images.

**Data Structures:**
The scene is represented by $N$ 2D Gaussian primitives. Each primitive stores attributes in 32-bit floating-point format [S1].
1.  **Geometry Attributes:**
    *   $\mu$: Center position (3 floats).
    *   $t_u, t_v$: Orientation vectors, represented as a quaternion (4 floats).
    *   $s_u, s_v$: Scale factors (2 floats).
    *   $o$: Opacity (1 float).
2.  **Shading Attributes:**
    *   $k_a$: Ambient coefficient (1 float).
    *   $k_d$: Diffuse coefficient (1 float).
    *   $k_s$: Specular coefficient (1 float).
    *   $\beta$: Shininess/exponent (1 float).
3.  **Texture Attribute:**
    *   Stored separately from individual splats. It is a global texture map shared by the scene.
    *   The total number of texels $T_{total}$ is fixed per basic scene [S1].

**Parameter Count:**
Given $N$ primitives, the total number of parameters for geometry and shading is calculated as [S1]:
$$ \text{# params} = 3N + 4N + N + N + N + N + N + N + N $$
Where:
*   $3N$ corresponds to $\mu$.
*   $4N$ corresponds to the quaternion for $\{t_u, t_v\}$.
*   The remaining $N$ terms correspond to $s_u, s_v, o, k_a, k_d, k_s, \beta$ (1 float each).

**Optimization Phases:**
The method proceeds in three sequential phases [S1]:
1.  **2DGS Initialization:** Initialize the geometry and basic appearance using 2DGS. This phase uses explicit ray-splat intersection for perspective-correct splatting and accurate geometry reconstruction [S1].
2.  **Shading Attributes Optimization:** Optimize $\{k_a, k_d, k_s, \beta\}$. This improves reconstruction accuracy by modeling view-dependent color variations [S1].
3.  **Texture Attribute Optimization:** Optimize the global texture map. This phase introduces a sparsity regularization term that encourages minimal texture values, which may slightly reduce reconstruction accuracy compared to the shading-only phase [S1]. Note: Spherical harmonic parameters and view-independent color $c_{ind}$ are used only in early phases and are not retrained in the final representation [S1].

**Scene Editing and Composition:**
*   **Text Prompt Refinement:** Users can iteratively apply new text prompts to an already stylized scene. The previous style pattern is partially preserved while being updated according to the latest prompt [S1].
*   **Multi-Scene Composition:** The framework accommodates multiple basic scenes (e.g., different transfer functions) in a single visualization. Each basic scene is edited independently and then seamlessly composed. This composition requires no additional optimization, incurring zero extra training costs [S1].
*   **Relighting:** Lighting direction is adjusted by modifying azimuthal and polar angles, achieving smooth and consistent relighting across the scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texgs_volvis_optimize(multi_view_images, text_prompts, num_primitives_target):
    """
    Optimizes a TexGS-VolVis scene representation.
    """
    # Phase 1: 2DGS Initialization
    # Initialize N primitives with geometry {mu, tu, tv, su, sv, o}
    # Use explicit ray-splat intersection for initialization
    primitives = initialize_2dgs(multi_view_images, num_primitives_target)
    
    # Phase 2: Shading Attributes Optimization
    # Optimize {ka, kd, ks, beta} for each primitive
    # This models view-dependent color variations
    shading_params = optimize_shading(primitives, multi_view_images)
    
    # Phase 3: Texture Attribute Optimization
    # Initialize global texture map with fixed size T_total
    T_total = 10_000_000  # Fixed at 1e7 texels per basic scene [S1]
    texture_map = initialize_texture(T_total)
    
    # Optimize texture map using text prompts and sparsity regularization
    # Sparsity regularization encourages minimal texture values [S1]
    texture_map = optimize_texture(primitives, shading_params, texture_map, 
                                   multi_view_images, text_prompts, 
                                   regularization='sparsity')
    
    return primitives, shading_params, texture_map

def edit_scene(primitives, shading_params, texture_map, new_text_prompt):
    """
    Iteratively refines the scene style using a new text prompt.
    """
    # Apply IP2P (Image-to-Image Prompt-to-Prompt) based on new_text_prompt
    # Partially preserve previous style, update according to new prompt [S1]
    updated_texture_map = apply_text_refinement(texture_map, new_text_prompt)
    
    # Note: Geometry is not frozen, allowing for geometry-consistent editing
    # without the elliptical artifacts seen in frozen-geometry baselines [S1]
    return primitives, shading_params, updated_texture_map

def compose_scenes(scene_list):
    """
    Composes multiple independently edited basic scenes.
    """
    # No additional optimization required for composition [S1]
    composed_scene = merge_scenes(scene_list)
    return composed_scene

def relight_scene(primitives, shading_params, texture_map, azimuth, polar):
    """
    Adjusts lighting direction using azimuthal and polar angles.
    """
    # Update lighting direction parameters
    # Achieves smooth and consistent relighting [S1]
    rendered_image = render(primitives, shading_params, texture_map, 
                            light_azimuth=azimuth, light_polar=polar)
    return rendered_image
```

**Edge Cases and Pitfalls:**
*   **Low-Texture Styles:** Flat or low-texture styles cause overly smoothed results, obscuring internal details [S1].
*   **Complex Structures:** Fragmented or complex structures lack continuous surfaces, preventing coherent style application [S1].
*   **Prompt Misinterpretation:** Text-driven editing relies on the pretrained IP2P model, which can misinterpret prompts (e.g., associating 'leaf beetle' with green instead of red/yellow/black) [S1].
*   **Rugged Surfaces:** Rugged or highly detailed surfaces (e.g., wood) are harder to fit than smooth ones (e.g., five jets) because they produce complex lighting effects in multi-view images, complicating geometry reconstruction [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The texture attribute dominates memory usage. For a fixed $T_{total} = 1 \times 10^7$ texels, the texture consumes ~114.5 MB per basic scene, regardless of the number of primitives $N$ [S1]. Geometry and shading attributes scale linearly with $N$.
    *   Example (Beetle dataset, 47,767 primitives): Geometry 1.8 MB, Shading 0.7 MB, Texture 114.5 MB, Total ~117 MB [S1]. Texture accounts for >95% of model size [S1].
*   **Time:** Fitting performance is fast. On the five jets dataset (56,836 primitives), fitting takes 0.8 minutes [S1].

**Measured Performance:**
*   **Five Jets Dataset:** PSNR 36.83 dB, SSIM 0.9889, Time 0.8 min, CPU/GPU Memory 10.0/5.6 GB, 56,836 primitives [S1].
*   **Other Datasets:** PSNR ranges from 24.58 dB (Ionization) to 36.83 dB (Five Jets). SSIM ranges from 0.8565 to 0.9889 [S1].

**Strengths and Limitations:**
*   **Strengths:** Decouples geometry and appearance, eliminating elliptical artifacts in text-driven NPSE while preserving visual expressiveness [S1]. Supports iterative text refinement and multi-scene composition with zero extra training cost [S1]. Uses 2DGS for accurate geometry reconstruction and perspective-correct splatting [S1].
*   **Limitations:** Texture memory usage is high and fixed per scene, making compression necessary for large-scale applications [S1]. Struggles with rugged surfaces and low-texture styles [S1]. Relies on external IP2P model for text interpretation, which can be unreliable [S1].

**When to Use:**
*   Use for text-driven non-photorealistic style editing of volume visualizations where geometry consistency is critical.
*   Use when multiple basic scenes (e.g., different transfer functions) need to be composed.
*   Do not use for scenes with highly fragmented structures or when low-texture styles are required, as results may be overly smoothed or incoherent.

## Background
TexGS-VolVis was introduced in the paper "TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting" [S1]. It builds upon 2D Gaussian Splatting (2DGS), which uses explicit ray-splat intersection for perspective-correct splatting and natively supports 2D texture maps, making it superior for geometry-consistent editing compared to standard 3DGS [S1]. The method addresses limitations in previous text-driven NPSE methods that either froze geometry (reducing expressiveness) or coupled geometry and appearance (causing artifacts) [S1].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
