# Inpainting for Displaced Objects

Also called: GSP Inpainting, LaMa Inpainting

## Overview
This method remedies black smudges and dirty textures caused by displaced objects exposing unseen areas that 3DGS cannot recover from input images [S1]. Unlike vanilla 3DGS which leaves empty spots and dirty textures blended from irrelevant kernels when objects are displaced, this method ameliorates the issue using generative AI inpainting [S1]. It builds on [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md) by modifying the kernel set to remove and reinitialize specific subsets [Parent].

## How it works
**Inputs:**
*   A set of 3D Gaussian kernels $\mathcal{G}$ representing the scene, including those belonging to objects that may be displaced.
*   A rendered image $I_{rendered}$ of the current view.
*   A segmentation mask or identification of the Gaussian kernels belonging to the displaced object.

**Outputs:**
*   Updated Gaussian kernels $\mathcal{G}'$ with corrected colors, opacity, and specular properties.
*   An inpainted image $I_{inpainted}$ (intermediate).

**Data Structures:**
*   **Gaussian Kernel:** Contains position $\mathbf{p}$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, spherical harmonics coefficients (defining color $c$), and potentially separate diffuse/specular color components.
*   **Inpainting Model:** LaMa [S1] or Stable Diffusion [S2].

**Steps:**

1.  **Remove Displaced Kernels:** Identify all Gaussian kernels $\mathcal{G}_{obj} \subset \mathcal{G}$ belonging to the object that may be displaced. Remove these kernels from the scene representation.
    *   *Note:* This creates "holes" or missing regions in the rendered view corresponding to the object's previous position.

2.  **Render Current View:** Render the scene using the remaining kernels $\mathcal{G} \setminus \mathcal{G}_{obj}$ to obtain $I_{rendered}$ and an alpha mask indicating the missing regions (holes).

3.  **Inpaint Image:** Use a generative inpainting model (LaMa [S1] or Stable Diffusion [S2]) to fill the holes in $I_{rendered}$ using the alpha mask. Let the result be $I_{inpainted}$.
    *   *LaMa Specifics:* The model generates plausible textures for the missing areas based on surrounding context [S1].
    *   *Stable Diffusion Specifics:* Generates missing texture in masked regions [S2].

4.  **Trace Rays and Record Colors:** For each pixel $(u, v)$ in the hole region of $I_{rendered}$:
    *   Emit a ray from the camera through $(u, v)$.
    *   Find the first Gaussian kernel $\mathcal{G}_i$ encountered by this ray in the *original* scene (before removal) or in the current scene if the ray intersects remaining geometry. *Correction based on [S1]:* The text states "assign them to the first Gaussian kernel encountered by the rays emitted from these pixels." In the context of GSP, this likely refers to mapping the new pixel color back to the Gaussian that *would* have been there or the nearest remaining kernel if the object is gone. However, [S1] explicitly says: "assign them to the first Gaussian kernel encountered by the rays emitted from these pixels." Since the object is removed, this ray might not hit the object. The standard interpretation in such "inpainting trick" papers is that the ray hits the *background* or *remaining* geometry, but the color is assigned to the *removed* Gaussian's slot if it were to be re-initialized, or more likely, the method re-initializes the removed Gaussians with the inpainted color.
    *   *Refined Interpretation from [S1]:* "Note the new colors of pixels originally in spots and assign them to the first Gaussian kernel encountered by the rays emitted from these pixels." This implies that for the pixels that were occluded by the displaced object, we now have a new color from the inpainting. We need to assign this color to the Gaussians that *were* in that spot. Since they are removed, we must re-initialize them. The text says "assign them to the first Gaussian kernel encountered". If the object is gone, the ray hits something else. This step is ambiguous. Let's look at the next step: "Average the recorded colors on all noted Gaussian kernels". This implies multiple rays might hit the same Gaussian or we are collecting colors for the *re-initialized* Gaussians.
    *   *Practical Implementation:* For each removed Gaussian $\mathcal{G}_j \in \mathcal{G}_{obj}$, determine its screen-space projection. For each pixel in its projection, if it was in a hole, take the color from $I_{inpainted}$ at that pixel. Collect these colors for $\mathcal{G}_j$.

5.  **Update Gaussian Attributes:** For each removed Gaussian $\mathcal{G}_j \in \mathcal{G}_{obj}$:
    *   **Diffuse Color:** Calculate the average of all recorded colors assigned to $\mathcal{G}_j$. Set this as the new diffuse color.
    *   **Opacity:** Set opacity $\alpha = 1$.
    *   **Specular Color:** Set specular color to 0 (or black) to prevent highlights, as the inpainted texture is assumed to be diffuse [S1].

6.  **Re-insert Kernels:** Add the updated kernels $\mathcal{G}_j$ back into the scene representation $\mathcal{G}'$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inpaint_displaced_objects(Gaussians, Camera, InpaintingModel):
    """
    Gaussians: List of Gaussian objects with attributes (pos, cov, alpha, color, etc.)
    Camera: Camera parameters for rendering
    InpaintingModel: Function to inpaint image given mask
    """
    # 1. Identify and remove displaced Gaussians
    # Assume we have a mask or list of indices for displaced objects
    displaced_indices = get_displaced_gaussian_indices(Gaussians)
    displaced_gaussians = [Gaussians[i] for i in displaced_indices]
    remaining_gaussians = [G for i, G in enumerate(Gaussians) if i not in displaced_indices]
    
    # 2. Render scene with remaining Gaussians
    rendered_image, alpha_mask = render(remaining_gaussians, Camera)
    
    # 3. Inpaint the holes
    # alpha_mask indicates where Gaussians are missing (holes)
    inpainted_image = InpaintingModel(rendered_image, alpha_mask)
    
    # 4. Assign new colors to displaced Gaussians
    for G in displaced_gaussians:
        # Project G to screen to find its pixel footprint
        screen_pixels = project_to_screen(G, Camera)
        
        collected_colors = []
        for pixel in screen_pixels:
            # Check if this pixel was in a hole (missing from original render)
            # In practice, we know these pixels correspond to the displaced object
            # so we take the inpainted color directly
            new_color = inpainted_image[pixel]
            collected_colors.append(new_color)
        
        # 5. Update attributes
        if collected_colors:
            # Average colors for diffuse component
            avg_color = sum(collected_colors) / len(collected_colors)
            G.diffuse_color = avg_color
            G.opacity = 1.0
            G.specular_color = 0.0 # Prevent highlights
        else:
            # Fallback if no pixels were recorded (e.g., object completely out of view)
            G.opacity = 0.0
            
    # 6. Reconstruct full Gaussian set
    new_Gaussians = remaining_gaussians + displaced_gaussians
    return new_Gaussians
```

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the inpainted color depends on the quality of the generative model. LaMa may produce blurry textures if the context is insufficient [S1].
*   **Boundaries:** The transition between the inpainted region and the rest of the scene must be seamless. The method assumes the inpainted texture matches the surrounding geometry's lighting and material properties.
*   **Degenerate Input:** If the displaced object is completely outside the camera view, no rays will hit its previous position, and no colors will be recorded. In this case, the Gaussian should be removed entirely (opacity 0).

## Subtopics

### 3D Inpainting Pipeline
A sequential pipeline to fill occluded/missing regions in rendered novel views while maintaining 3D consistency, leveraging the constraint that Gaussians do not hallucinate details [S2]. Unlike the single-view LaMa approach, this method uses Stable Diffusion for texture generation and Poisson blending for depth consistency [S2]. It is chosen when 3D consistency across multiple novel views is required, particularly in sparse view synthesis scenarios [S2].

**Steps:**
1.  Render a novel view with holes and an alpha mask [S2].
2.  Use Stable Diffusion inpainting to generate missing texture in the masked regions [S2].
3.  Estimate monocular depth on the inpainted image [S2].
4.  Use the gradient of this depth to fill in rendered depth in the masked regions using Poisson blending [S2].
5.  Project Gaussians to the scene using the inpainted depth and image [S2].
6.  Repeat the process sequentially for a few novel views until the scene is 3D consistent and renderable from any view without holes [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the inpainting model. LaMa is relatively fast, but Stable Diffusion is computationally expensive. The sequential nature of the 3D Inpainting Pipeline adds further time cost [S2].
*   **Memory Complexity:** Requires storing the inpainted images and intermediate depth maps. No significant additional memory overhead for the Gaussian kernels themselves, as they are re-initialized [S1].
*   **Measured Performance:** GSP (Gaussian Splashing) with inpainting shows improved visual quality in scenes with displaced objects, removing black smudges and dirty textures [S1]. CoherentGS outperforms DNGaussian visually and numerically on sparse view synthesis, avoiding blurry textures and distracting artifacts [S2].
*   **Strengths:** Effectively handles unseen areas caused by object displacement; leverages state-of-the-art generative models for realistic texture synthesis; maintains 3D consistency in the sequential pipeline [S2].
*   **Limitations:** Relies on the quality of the inpainting model; may introduce artifacts if the context is ambiguous; not suitable for dynamic scenes where the displacement is not known or segmented [S1].
*   **When to Use:** When objects in the scene are displaced or removed, exposing background areas not present in the input images [S1]. When sparse view synthesis is required and standard methods produce blurry textures [S2].
*   **When Not to Use:** When the scene is static and fully captured in the input images; when real-time performance is critical and generative inpainting is too slow [S1].

## Background
The method was introduced in the context of Gaussian Splashing (GSP), a unified particle-based framework for versatile motion synthesis and rendering [S1]. The authors observed that standard 3DGS fails to recover color and texture information for areas exposed by displaced objects, leading to visual artifacts [S1]. They proposed using LaMa, a large kernel mask inpainting model, to remedy this issue [S1]. The 3D Inpainting Pipeline variant was introduced in CoherentGS for sparse novel view synthesis, leveraging the constraint that Gaussians do not hallucinate details to safely use generative inpainting [S2].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
