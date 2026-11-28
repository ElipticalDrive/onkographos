# Multi-view SDS

Also called: BoostDream

## Overview
Multi-view SDS is a plug-and-play regularization method that addresses the "Janus problem" (inconsistent facial features or object orientations) inherent in standard Single-view Score Distillation Sampling (SDS) by enforcing consistency across multiple viewpoints simultaneously [S1][S2, unofficial]. Unlike standard SDS which optimizes one view at a time, this method stitches rendered images from multiple sampled views into a single composite image to compute gradients under multi-view normal map conditions, thereby improving the geometric coherence of the generated 3D asset [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A differentiable 3D representation (NeRF, 3DGS, or DMTet) [S1][S2, unofficial]; a text prompt; a diffusion model (e.g., Stable Diffusion) [S1][S2, unofficial]; a set of $N$ camera viewpoints (typically $N=4$) [S1][S2, unofficial].
*   **Outputs:** Optimized 3D representation parameters [S1][S2, unofficial].

**Data Structures:**
*   **Composite Image:** A 2D image formed by stitching $N$ rendered views. For $N=4$, the layout is a $2 \times 2$ grid [S1][S2, unofficial].
*   **Multi-view Normal Map:** A composite image of normal vectors corresponding to the stitched views, used to condition the gradient calculation [S1][S2, unofficial].

**Steps:**
1.  **View Sampling:** Sample $N$ distinct camera viewpoints (e.g., 4 views) for the current optimization step [S1][S2, unofficial].
2.  **Rendering:** Render the current 3D representation from each of the $N$ viewpoints to produce individual color images $\mathbf{I}_1, \dots, \mathbf{I}_N$ and normal maps $\mathbf{N}_1, \dots, \mathbf{N}_N$ [S1][S2, unofficial].
3.  **Stitching:** Construct a composite image $\mathbf{I}_{multi}$ by stitching $\mathbf{I}_1, \dots, \mathbf{I}_N$ into a $2 \times 2$ grid (for $N=4$) [S1][S2, unofficial]. Similarly, construct a composite normal map $\mathbf{N}_{multi}$ from $\mathbf{N}_1, \dots, \mathbf{N}_N$ [S1][S2, unofficial].
4.  **Diffusion Conditioning:** Input $\mathbf{I}_{multi}$ and $\mathbf{N}_{multi}$ into the pre-trained 2D diffusion model [S1][S2, unofficial].
5.  **Gradient Calculation:** Compute the SDS gradients using the diffusion model's output conditioned on the multi-view inputs [S1][S2, unofficial]. The gradients are calculated under the condition of the multi-view normal map [S1][S2, unofficial].
6.  **Optimization:** Backpropagate the gradients through the differentiable rendering process to update the 3D representation parameters (density $\sigma$, color $\mathbf{c}$, or Gaussian attributes) [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_view_sds_step(nerf_model, diffusion_model, text_prompt, cameras):
    """
    Perform one optimization step for Multi-view SDS.
    
    Args:
        nerf_model: Differentiable 3D representation (NeRF/3DGS/DMTet).
        diffusion_model: Pre-trained 2D diffusion model.
        text_prompt: Text string for generation.
        cameras: List of N camera extrinsics/intrinsics (e.g., N=4).
        
    Returns:
        Loss value and gradients for nerf_model parameters.
    """
    N = len(cameras)
    rendered_colors = []
    rendered_normals = []
    
    # Step 1 & 2: Render from N views
    for cam in cameras:
        color_img, normal_img = nerf_model.render(cam)
        rendered_colors.append(color_img)
        rendered_normals.append(normal_img)
        
    # Step 3: Stitch into 2x2 grid (assuming N=4)
    # Implementation detail: Resize images to match height/width if necessary
    # and arrange in a 2x2 block structure.
    composite_color = stitch_images_2x2(rendered_colors)
    composite_normal = stitch_images_2x2(rendered_normals)
    
    # Step 4 & 5: Compute SDS gradients with multi-view conditioning
    # The diffusion model takes the composite color and normal map as input.
    # Standard SDS formula applied to the composite image.
    # Note: Exact diffusion conditioning mechanism (e.g., ControlNet weights) 
    # is not specified in sources [S1][S2, unofficial].
    gradients = compute_sds_gradients(
        diffusion_model=diffusion_model,
        image=composite_color,
        normal_map=composite_normal,
        text_prompt=text_prompt
    )
    
    # Step 6: Backpropagate to update 3D representation
    # Gradients flow from the composite image loss back through the stitching 
    # and rendering operations to the 3D model parameters.
    loss = compute_loss(gradients)
    return loss
```

**Parameters:**
*   **Number of views ($N$):** Typically 4 [S1][S2, unofficial].
*   **Grid Layout:** $2 \times 2$ [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Resolution Mismatch:** Stitching requires rendered views to have compatible dimensions; resizing may introduce artifacts [S1][S2, unofficial].
*   **Normal Map Consistency:** The multi-view normal map must be correctly aligned with the color images to provide valid geometric conditioning [S1][S2, unofficial].

## Subtopics

### BoostDream
The specific implementation of Multi-view SDS that introduced the 2x2 stitching technique [S1][S2, unofficial].

### GaussianDiffusion
A related method that incorporates structured noise from multiple viewpoints to alleviate the Janus problem, using variational 3DGS [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Increases optimization time compared to single-view SDS due to rendering and processing $N$ views per step [S1].
*   **Memory Complexity:** Higher memory usage due to storing intermediate renders for $N$ views and the composite image [S1][S2, unofficial].
*   **Strengths:** Effectively reduces the Janus problem; improves geometric coherence; plug-and-play with various 3D representations (NeRF, 3DGS, DMTet) [S1][S2, unofficial].
*   **Limitations:** Increased computational cost; potential for artifacts if stitching or normal map conditioning is poorly implemented [S1][S2, unofficial].
*   **When to use:** When generating 3D assets with distinct front/back asymmetry (e.g., faces, characters) where single-view SDS fails to maintain consistency [S1][S2, unofficial].

## Background
Multi-view SDS was proposed by BoostDream [S1][S2, unofficial]. It addresses limitations in standard SDS optimization, where views are sampled one by one, leading to inconsistencies [S1][S2, unofficial]. The method leverages the efficiency of 3D Gaussian Splatting (3DGS) in some implementations, replacing MipNeRF as used in DreamGaussian, to render high-resolution images with high FPS and small memory [S1]. It is part of a broader trend of improving 3D generation by incorporating multi-view supervision [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
