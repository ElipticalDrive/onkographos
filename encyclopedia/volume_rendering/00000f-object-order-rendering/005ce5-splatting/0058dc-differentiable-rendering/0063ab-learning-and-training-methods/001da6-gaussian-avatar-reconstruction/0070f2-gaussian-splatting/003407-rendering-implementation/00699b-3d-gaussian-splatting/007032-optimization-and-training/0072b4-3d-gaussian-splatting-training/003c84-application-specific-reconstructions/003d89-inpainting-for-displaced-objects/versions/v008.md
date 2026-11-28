# Inpainting for Displaced Objects

Also called: GSP Inpainting, LaMa Inpainting

## Overview
Inpainting for Displaced Objects remedies the visual artifacts—specifically black smudges and dirty textures—that arise in 3D Gaussian Splatting when an object moves, exposing areas previously occluded and unseen by the input cameras [S1]. Unlike vanilla 3DGS, which leaves empty spots or blends irrelevant kernel colors into these voids, this method uses generative AI to fill the missing texture and reassigns the resulting pixel colors to the Gaussian kernels [S1]. It builds on [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md) by introducing a post-processing or auxiliary step that decouples texture recovery from the standard differentiable rendering loop for displaced entities.

## How it works
This method addresses the issue where displaced objects reveal background or unseen surfaces that were not present in the training images, causing the renderer to produce artifacts because the Gaussian kernels for those areas lack valid color data [S1].

**Inputs:**
*   A set of 3D Gaussians $\mathcal{G}$ representing the scene, including those belonging to the object expected to displace.
*   A set of training images or a target novel view where the object has moved.
*   A segmentation mask identifying the Gaussian kernels belonging to the displaced object.
*   An inpainting model (e.g., LaMa [S1] or Stable Diffusion [S2]).

**Outputs:**
*   Updated Gaussian kernels with corrected diffuse colors, opacity, and specular properties for the displaced object.

**Data Structures:**
*   **Gaussian Kernel:** Standard 3DGS structure containing position $\mathbf{x}$, scale $\mathbf{s}$, rotation $\mathbf{q}$, opacity $\alpha$, and color coefficients (SH or RGB). For this method, we specifically track diffuse color $c_{diff}$ and specular color $c_{spec}$.
*   **Ray Buffer:** A temporary storage for rays emitted from camera pixels that fall into the "holes" or displaced regions.

**Steps:**

1.  **Identify and Remove Displaced Kernels:**
    Identify all Gaussian kernels $\mathcal{G}_{obj} \subset \mathcal{G}$ that belong to the object capable of displacement. Remove these kernels from the active rendering set to create a "hole" in the scene representation where the object was previously located or where it moves to [S1].

2.  **Render the Scene:**
    Render the scene using the remaining Gaussians $\mathcal{G} \setminus \mathcal{G}_{obj}$ to generate an image $I_{rendered}$ and an alpha mask $M$ indicating the missing regions (where the object should be) [S1, S2].

3.  **Inpaint the Image:**
    Use a generative inpainting model to fill the masked regions in $I_{rendered}$.
    *   For GSP/LaMa: Use LaMa [61] to inpaint the rendered results [S1].
    *   For CoherentGS: Use Stable Diffusion inpainting [42] to generate missing texture in the masked regions [S2].
    Let the inpainted image be $I_{inpaint}$.

4.  **Trace Rays and Record Colors (GSP Method):**
    For each pixel $(u, v)$ in the masked region $M$:
    a. Emit a ray from the camera through $(u, v)$.
    b. Find the first Gaussian kernel $\mathcal{G}_i$ encountered by this ray that was originally part of $\mathcal{G}_{obj}$ (or is projected to this location in the new view) [S1].
    c. Record the new color $C_{new}(u, v)$ from $I_{inpaint}$ at $(u, v)$.
    d. Associate $C_{new}(u, v)$ with $\mathcal{G}_i$ [S1].

5.  **Aggregate Colors:**
    For each Gaussian kernel $\mathcal{G}_i$ in the displaced object:
    a. Collect all recorded colors $C_{new}$ associated with it from the previous step.
    b. Compute the average diffuse color:
       $$ c_{diff} = \frac{1}{N} \sum_{j=1}^{N} C_{new, j} $$
       where $N$ is the number of rays/pixels associated with $\mathcal{G}_i$ [S1].

6.  **Update Gaussian Properties:**
    For each kernel $\mathcal{G}_i$ in $\mathcal{G}_{obj}$:
    a. Set diffuse color to $c_{diff}$.
    b. Set opacity $\alpha = 1$ [S1].
    c. Set specular color $c_{spec} = 0$ to prevent highlights in the inpainted regions [S1].

**Alternative Pipeline (CoherentGS 3D Inpainting):**
If maintaining 3D consistency across multiple novel views is required, use the sequential pipeline [S2]:
1.  Render a novel view with holes and an alpha mask.
2.  Use Stable Diffusion inpainting to generate missing texture.
3.  Estimate monocular depth on the inpainted image.
4.  Use the gradient of this depth to fill in rendered depth in the masked regions using Poisson blending [38].
5.  Project Gaussians to the scene using the inpainted depth and image.
6.  Repeat sequentially for a few novel views until the scene is 3D consistent and renderable from any view without holes [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** The assignment of colors to the "first Gaussian kernel encountered" requires accurate ray-Gaussian intersection tests. If the ray misses all kernels in the displaced region, the color is lost [S1].
*   **Hallucination:** While generative inpainting fills holes, it may hallucinate details not present in the 3D structure. CoherentGS notes that constraining Gaussian movement prevents this hallucination from corrupting the 3D geometry, allowing safe use of 2D inpainting [S2].
*   **Specular Artifacts:** Failing to set specular color to 0 may result in incorrect highlights on the inpainted surface, as the generative model only provides diffuse texture [S1].

## Subtopics

### 3D Inpainting Pipeline
A sequential pipeline designed to fill occluded or missing regions in rendered novel views while maintaining 3D consistency [S2]. Unlike the single-view GSP method, this approach leverages the constraint that Gaussians do not hallucinate details in occluded areas, allowing the use of state-of-the-art 2D inpainting tools like Stable Diffusion [S2]. It involves rendering a view with holes, inpainting the texture, estimating monocular depth, and using Poisson blending to integrate the depth gradient into the rendered depth before re-projecting Gaussians [S2]. This process is repeated across multiple novel views until the scene is fully 3D consistent and hole-free [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the inpainting model. LaMa is generally faster than Stable Diffusion [S1, S2]. The ray tracing and color aggregation steps are linear with respect to the number of pixels in the masked region and the number of displaced Gaussians.
*   **Memory Complexity:** Requires storing the inpainted image and temporary ray buffers. No significant additional memory overhead for the Gaussian structure itself, as properties are updated in place.
*   **Performance:** Effectively eliminates black smudges and dirty textures caused by displaced objects [S1]. It ameliorates issues that vanilla 3DGS cannot recover from input images [S1].
*   **Strengths:** Simple to implement as a post-processing or auxiliary step; leverages mature 2D generative models; does not require modifying the core differentiable rendering engine [S1, S2].
*   **Limitations:** The inpainted texture is 2D and may not perfectly match the 3D geometry if the displacement is large or complex; requires segmentation masks for the displaced objects [S1].
*   **When to use:** When objects in the scene undergo significant displacement, exposing previously occluded background or unseen surfaces [S1].
*   **When not to use:** When the scene is static or displacements are minimal and recoverable by standard densification [S1].

## Background
The method was introduced in "Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering" (2025) by Law et al. [S1]. It addresses a specific failure mode of 3D Gaussian Splatting where dynamic objects leave visual artifacts when they move. The technique utilizes LaMa [61] for fast inpainting. A variant using Stable Diffusion and depth consistency was later proposed in "CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians" (2025) [S2].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
