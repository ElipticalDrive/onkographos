# CLIP-NeRF


## Overview
CLIP-NeRF is a neural radiance field method that enables text- or image-guided editing of 3D scenes by extracting latent space displacements from user input via the CLIP model [S1, S2]. It differs from prior editing methods like Edit-NeRF by bypassing per-edit latent code optimization, achieving significant speedups (8–60x) through a disentangled conditional architecture that maps semantic input directly to shape and appearance modifications [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF representation (initial weights), a user-provided text prompt or reference image, and camera parameters for the target view.
*   **Outputs:** A modified radiance field capable of rendering the scene according to the semantic guidance, and the rendered image.

**Data Structures:**
The method relies on a disentangled conditional NeRF architecture [S2]. Key components include:
1.  **Base NeRF:** Standard MLPs for density $\sigma$ and color $\mathbf{c}$, conditioned on latent codes.
2.  **Shape Latent Code ($z_s$):** A vector controlling geometric deformation.
3.  **Appearance Latent Code ($z_a$ or $z_c$):** A vector controlling texture and color [S1, S2].
4.  **Shape Mapper Network:** A neural network that takes the CLIP embedding of the input text/image and outputs the displacement for $z_s$ [S1].
5.  **Appearance Mapper Network:** A neural network that takes the CLIP embedding and outputs the displacement for $z_a$ [S1].
6.  **Deformation Network (Instance-Specific Network):** An MLP that takes the encoded position and the shape latent code to produce a deformation vector, modifying the input position before it enters the density network [S1, S2].

**Steps:**
1.  **CLIP Embedding:** Encode the user input (text or image) into a latent vector using the pre-trained CLIP model [S1, S2].
2.  **Latent Displacement Extraction:** Pass the CLIP embedding through the Shape Mapper and Appearance Mapper networks to compute displacements $\Delta z_s$ and $\Delta z_a$ [S1].
3.  **Latent Code Update:** Update the scene's latent codes: $z_s^{new} = z_s^{base} + \Delta z_s$ and $z_a^{new} = z_a^{base} + \Delta z_a$ [S1].
4.  **Deformation:** For a ray point $\mathbf{x}$, encode it with positional encoding $\gamma(\mathbf{x})$. Pass $\gamma(\mathbf{x})$ and $z_s^{new}$ through the Deformation Network to get a deformation vector $\mathbf{d}_{def}$ [S1, S2]. Compute the deformed position $\mathbf{x}' = \mathbf{x} + \mathbf{d}_{def}$ [S1].
5.  **Radiance Field Query:**
    *   Query the Density Network with $\gamma(\mathbf{x}')$ and $z_s^{new}$ to get density $\sigma$ [S1, S2].
    *   Query the Color Network with $\gamma(\mathbf{x}')$, viewing direction $\mathbf{d}$, and $z_a^{new}$ to get color $\mathbf{c}$ [S1, S2].
6.  **Volume Rendering:** Render the pixel color $C(\mathbf{r})$ using the standard volumetric rendering integral [S2]:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clip_nerf_render(ray_origin, ray_direction, clip_input, base_nerf_weights, 
                     shape_mapper_weights, appearance_mapper_weights, 
                     deformation_net_weights, density_net_weights, color_net_weights):
    # 1. Get CLIP embedding
    clip_embedding = get_clip_embedding(clip_input)
    
    # 2. Extract latent displacements
    delta_z_s = shape_mapper(clip_embedding, shape_mapper_weights)
    delta_z_a = appearance_mapper(clip_embedding, appearance_mapper_weights)
    
    # 3. Update latent codes (assuming base codes are stored or initialized)
    z_s = base_nerf_weights['z_s'] + delta_z_s
    z_a = base_nerf_weights['z_a'] + delta_z_a
    
    # 4. Sample points along ray
    t_vals = sample_t_values(ray_origin, ray_direction, num_samples)
    points = ray_origin + ray_direction * t_vals
    
    # 5. Encode positions
    encoded_points = positional_encoding(points)
    
    # 6. Deform points
    # Deformation network takes encoded position and shape latent
    deformation_vectors = deformation_net(encoded_points, z_s, deformation_net_weights)
    deformed_points = points + deformation_vectors
    
    # 7. Query Density
    # Density network takes deformed encoded position and shape latent
    densities = density_net(deformed_points, z_s, density_net_weights)
    
    # 8. Query Color
    # Color network takes deformed encoded position, viewing direction, and appearance latent
    colors = color_net(deformed_points, ray_direction, z_a, color_net_weights)
    
    # 9. Volume Rendering
    alphas = 1.0 - torch.exp(-densities * t_diffs)
    transmittances = torch.cumprod(torch.exp(-alphas), dim=0)
    weights = alphas * transmittances
    rendered_color = torch.sum(weights * colors, dim=0)
    
    return rendered_color
```

**Parameters:**
*   **Number of frequency bands ($L$):** Not given in the sources.
*   **Deformation Network Depth/Width:** Not given in the sources.
*   **Mapper Network Architecture:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Semantic Ambiguity:** CLIP embeddings may not perfectly capture 3D geometric intent, leading to unnatural deformations if the text prompt is vague [S2].
*   **Identity Preservation:** The method relies on the base NeRF's ability to generalize; poor base training may result in artifacts during editing [S1].

## Subtopics

### Edit-NeRF
The predecessor method that optimizes latent codes per edit, which CLIP-NeRF replaces with direct mapping for speed. [Edit-NeRF](8663-edit-nerf.md)

### Deformable NeRF
Uses deformation networks to model non-rigid scenes; CLIP-NeRF adapts this for semantic editing [S1]. [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md)

## Operating characteristics
*   **Time Complexity:** Significantly faster than Edit-NeRF during the editing phase because it avoids iterative optimization of latent codes for each edit [S1]. The speedup is reported as a factor of ~8–60 depending on the task [S1].
*   **Memory Complexity:** Requires storing the base NeRF weights, the mapper networks, and the deformation network [S1, S2].
*   **Strengths:** Enables global deformation and shape operations with high freedom; fast editing without per-edit optimization [S2].
*   **Limitations:** Quality depends on the alignment between CLIP's semantic space and 3D geometry; may struggle with fine-grained local edits compared to optimization-based methods [S1, S2].

## Background
CLIP-NeRF was proposed by Wang et al. in 2022 [S2]. It builds upon the latent-conditioned NeRF framework and Edit-NeRF, aiming to overcome the slow editing speed of iterative latent optimization by leveraging the pre-trained multimodal capabilities of CLIP [S1, S2]. It is used for text- or image-guided 3D scene editing and generation [S1, S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
