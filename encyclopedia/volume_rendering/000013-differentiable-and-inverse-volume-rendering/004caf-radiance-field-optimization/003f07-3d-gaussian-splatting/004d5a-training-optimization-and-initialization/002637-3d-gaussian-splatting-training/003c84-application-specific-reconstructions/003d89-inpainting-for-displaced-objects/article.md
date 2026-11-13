# Inpainting for Displaced Objects

Also called: GSP Inpainting, LaMa Inpainting

## Overview
This method remedies black smudges and dirty textures caused by displaced objects exposing unseen areas that 3D Gaussian Splatting (3DGS) cannot recover from input images [S1]. Unlike vanilla 3DGS, which leaves empty spots and dirty textures blended from irrelevant kernels when objects are displaced, this technique ameliorates the issue using generative AI inpainting [S1]. It is specifically designed for scenarios where object movement reveals background or occluded regions not present in the training data.

## How it works
**Inputs:**
*   A set of 3D Gaussian kernels representing the scene, including those belonging to the displaced object.
*   A rendered image of the scene containing artifacts (black smudges/dirty textures) in the regions exposed by the displacement.
*   A generative inpainting model (specifically LaMa [61] in the primary formulation) [S1].

**Outputs:**
*   Updated Gaussian kernels with corrected diffuse colors, opacity, and specular properties.
*   A clean rendered image free of displacement artifacts.

**Data Structures:**
*   **Gaussian Kernel:** Standard 3DGS attributes including position $\mathbf{p}$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, spherical harmonics coefficients for color, and diffuse/specular color components.
*   **Rendered Image:** A 2D array of pixel colors and an alpha mask indicating valid/invalid regions.

**Steps:**
1.  **Kernel Removal:** Identify and remove all Gaussian kernels associated with the object that may be displaced. This clears the "dirty" or missing texture data from the scene representation [S1].
2.  **Rendering:** Render the scene with the displaced object's kernels removed. This results in an image with holes or background regions where the object was previously located.
3.  **Generative Inpainting:** Use LaMa [61] to inpaint the rendered results. The model generates plausible texture for the missing regions based on the surrounding context [S1].
4.  **Color Tracking:** Note the new colors of pixels in the inpainted regions. For each pixel in the inpainted area, cast a ray from the camera through that pixel and identify the first Gaussian kernel encountered by this ray [S1].
5.  **Color Assignment:** Assign the new inpainted color to the identified Gaussian kernel. If multiple rays (from different views or iterations) map to the same kernel, average the recorded colors on all noted Gaussian kernels to determine their final diffuse color [S1].
6.  **Parameter Update:**
    *   Set the opacity $\alpha$ of the affected kernels to 1 [S1].
    *   Set the specular color to 0 to prevent highlights in the inpainted regions [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inpaint_displaced_objects(gaussians, camera, la_ma_model):
    # Step 1: Identify and remove kernels of the displaced object
    # Assume 'displaced_object_mask' identifies which kernels belong to the object
    active_gaussians = [g for g in gaussians if not displaced_object_mask[g.id]]
    
    # Step 2: Render the scene without the object
    # Returns image with holes/background and ray data
    rendered_image, ray_data = render_scene(active_gaussians, camera)
    
    # Step 3: Inpaint the rendered image using LaMa
    # LaMa fills in the missing regions with generated texture
    inpainted_image = la_ma_model.inpaint(rendered_image, ray_data.alpha_mask)
    
    # Step 4 & 5: Assign new colors to kernels
    # For each pixel in the inpainted region, find the corresponding kernel
    for pixel in inpainted_image.masked_regions():
        ray = camera.get_ray(pixel.u, pixel.v)
        # Find the first kernel intersected by the ray
        kernel = ray.cast_first_intersection(active_gaussians)
        
        if kernel is not None:
            # Record the new color from the inpainted image
            new_color = inpainted_image.get_color(pixel.u, pixel.v)
            kernel.color_history.append(new_color)
            
    # Average the recorded colors for diffuse color
    for kernel in active_gaussians:
        if kernel.color_history:
            kernel.diffuse_color = average(kernel.color_history)
            
    # Step 6: Update parameters
    for kernel in active_gaussians:
        kernel.opacity = 1.0
        kernel.specular_color = 0.0
        
    return active_gaussians
```

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of color assignment depends on the precision of ray-kernel intersection tests.
*   **Boundaries:** The method assumes the displaced object is fully segmented or identifiable. If the mask is inaccurate, kernels may be incorrectly removed or retained.
*   **Degenerate Input:** If the inpainted region is too large or lacks sufficient context, LaMa may generate hallucinated details that do not match the scene's lighting or geometry.

## Subtopics

### 3D Inpainting Pipeline
A sequential pipeline to fill occluded/missing regions in rendered novel views while maintaining 3D consistency, leveraging the constraint that Gaussians do not hallucinate details [S2]. Unlike the single-view LaMa approach, this method iterates across multiple views. Steps: 1. Render a novel view with holes and an alpha mask. 2. Use Stable Diffusion inpainting to generate missing texture in the masked regions. 3. Estimate monocular depth on the inpainted image. 4. Use the gradient of this depth to fill in rendered depth in the masked regions using Poisson blending. 5. Project Gaussians to the scene using the inpainted depth and image. 6. Repeat the process sequentially for a few novel views until the scene is 3D consistent and renderable from any view without holes [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the cost of rendering and the generative inpainting model. LaMa and Stable Diffusion are computationally expensive compared to standard rasterization.
*   **Memory Complexity:** Requires storing the generative model weights and intermediate rendered images.
*   **Performance:** Significantly reduces artifacts in dynamic scenes with displaced objects compared to vanilla 3DGS [S1].
*   **Strengths:** Effectively handles unseen background regions revealed by object movement. Prevents black smudges and dirty textures.
*   **Limitations:** Relies on the quality of the generative inpainting model. May introduce inconsistencies if the inpainted texture does not match the lighting or geometry of the rest of the scene. Not suitable for reconstructing geometry that was never visible in any input view.
*   **When to use:** When objects in a scene move and expose previously occluded background areas.
*   **When not to use:** When the goal is to reconstruct the object itself, or when the occluded regions contain complex geometry that cannot be inferred from surrounding context.

## Background
Originated in the context of Gaussian Splashing (GSP), a method for unified particle-based motion synthesis and rendering [S1]. The technique was introduced to address the limitation of 3DGS in handling dynamic scenes where objects displace and reveal unseen areas. The primary source cites LaMa [61] as the inpainting model used. A variant using Stable Diffusion and Poisson blending for 3D consistency was later proposed in CoherentGS [S2].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
