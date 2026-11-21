# Color-Geometry Dependency Assumption

Also called: Photometric Consistency Optimization for Shape, Shape-from-Shading

## Overview
The Color-Geometry Dependency Assumption leverages the correlation between surface texture and geometry (normals) to refine 3D reconstruction, effectively applying shape-from-shading principles to neural fields [S1, unofficial]. It alleviates texture-induced artifacts by enforcing multi-view consistency during the joint training of geometry and color fields [S1, unofficial]. Unlike standard photometric losses that treat color as an independent attribute, this method assumes that shading cues provide geometric information, which can improve surface detail but may introduce relief-like artifacts on textureless or decorrelated surfaces [S1, unofficial].

## How it works
The method integrates shading-based regularization into the optimization of neural radiance fields or implicit surfaces. It relies on the differentiability of the rendering operator to backpropagate gradients from the photometric error through the shading model to the geometry parameters [44-radiance-field-optimization.md].

**Inputs and Outputs:**
*   **Inputs:** Camera poses, 2D anchor images, initial geometry field (e.g., SDF), and color field (e.g., RGB or albedo/shading decomposition).
*   **Outputs:** Refined geometry (normals/SDF) and color fields that are consistent with the observed shading in the input images.

**Data Structures:**
*   **Geometry Field:** Typically an implicit representation (e.g., SDF) or explicit voxel grid storing density/occupancy.
*   **Color Field:** Stores RGB values or decomposed components (Albedo $\mathbf{a}$ and Shading $\mathbf{s}$).
*   **Normals:** Computed as the gradient of the SDF: $\mathbf{n} = \nabla \phi(\mathbf{x}) / \|\nabla \phi(\mathbf{x})\|$ [S1, unofficial].

**Steps:**
1.  **Ray Sampling:** For each pixel, sample points along the ray $\mathbf{r}$ using the current geometry field.
2.  **Rendering:** Compute the rendered color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model [44-radiance-field-optimization.md]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\mathbf{c}_i$ is the color at sample $i$. If decomposed, $\mathbf{c}_i = \mathbf{a}_i \odot \mathbf{s}_i$.
3.  **Shading Computation:** Estimate the shading $\mathbf{s}_i$ at each sample point based on the surface normal $\mathbf{n}_i$ and a lighting model (e.g., diffuse reflection with a light source direction $\mathbf{l}$):
    $$
    \mathbf{s}_i = \max(0, \mathbf{n}_i \cdot \mathbf{l})
    $$
    [S1, unofficial]
4.  **Loss Calculation:** Compute the photometric loss between the rendered color $\hat{C}(\mathbf{r})$ and the ground truth image color $C(\mathbf{r})$. The gradient of this loss flows back to the color field and, through the shading dependency, to the normals and geometry.
5.  **Regularization (Optional):** Apply color regularization to the shading field to reduce artifacts while maintaining color fidelity [S2]. This is often implemented as a smoothness constraint on the shading field [S2].

**Parameters:**
*   **Lighting Model:** Diffuse, Phong, etc. [S1, unofficial].
*   **Regularization Weight:** Controls the strength of the color regularization term [S2].

**Edge Cases and Pitfalls:**
*   **Texture-Geometry Decoupling:** If surface texture is not correlated with geometry (e.g., painted patterns on a flat surface), the assumption fails, leading to "relief-like" artifacts where texture is interpreted as geometry [S1, unofficial].
*   **Specular Highlights:** Strong specularities can violate diffuse shading assumptions, causing geometric errors.
*   **Lighting Estimation:** Incorrect lighting direction or intensity assumptions can lead to incorrect normal estimates.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_color_geometry_loss(sdf, albedo_field, shading_field, camera, gt_image):
    """
    Computes the photometric loss assuming color-geometry dependency.
    """
    # 1. Sample points along rays
    points, weights = sample_rays(sdf, camera)
    
    # 2. Compute normals from SDF
    normals = compute_normals(sdf, points)
    
    # 3. Estimate shading based on normals and assumed lighting
    # Assuming a simple diffuse model with a fixed light direction L
    L = normalize(torch.tensor([0.0, 0.0, 1.0])) # Example light direction
    shading = torch.clamp(torch.sum(normals * L, dim=-1, keepdim=True), 0.0, 1.0)
    
    # 4. Compute rendered color
    # Albedo is retrieved from the albedo field
    albedos = albedo_field(points)
    colors = albedos * shading
    
    # 5. Composite using weights (simplified)
    rendered_color = torch.sum(colors * weights, dim=0)
    
    # 6. Compute photometric loss
    loss = torch.mean((rendered_color - gt_image) ** 2)
    
    return loss
```

## Subtopics

### Color Regularization
A regularization technique applied to the shading field to produce cleaner reconstructions with fewer artifacts while maintaining color [S2]. It does not degrade reconstruction quality (metrics remain essentially unchanged) but qualitatively improves the shading field [S2]. On synthetic scenes, it yields a Shading PSNR of 22.876 compared to 22.957 without it, indicating negligible metric change but qualitative improvement [S2].

## Operating characteristics
*   **Strengths:** Enhances surface detail and geometric accuracy by leveraging shading cues [S1, unofficial]. Reduces artifacts in the shading field when combined with color regularization [S2].
*   **Limitations:** Prone to artifacts when texture and geometry are not correlated (e.g., textureless surfaces with painted patterns) [S1, unofficial]. Requires careful handling of lighting assumptions.
*   **Performance:** Training time may increase slightly due to the additional shading computation and regularization terms (e.g., DVGO + NeuS w/ smoothness increased from 12min to 15min in one study) [S1, unofficial].

## Background
The assumption is rooted in the shape-from-shading technique, which has been proven effective in surface reconstruction [S1, unofficial]. It builds on earlier neural surface reconstruction methods like NeuS (Yariv et al., 2020, 2021) and others (Oechsle et al., 2021) that implicitly or explicitly use shading information [S1, unofficial]. The method is commonly used in neural radiance field optimization and neural surface reconstruction to improve geometric fidelity [S1, unofficial][S2].

## Sources
[S1] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
[S2] Intrinsic decomposition and editing of 3D Gaussian splats, https://doi.org/10.1145/3804495, fetched 2026-10-27
