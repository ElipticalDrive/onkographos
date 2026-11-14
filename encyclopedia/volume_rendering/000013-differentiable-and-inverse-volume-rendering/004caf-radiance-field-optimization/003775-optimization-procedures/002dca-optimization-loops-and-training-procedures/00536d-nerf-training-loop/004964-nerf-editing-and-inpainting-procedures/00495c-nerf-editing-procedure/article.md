# NeRF Editing Procedure

Also called: Feature Field Distillation Editing, DFF-based Editing

## Overview
This procedure enables direct 3D editing of a pre-trained NeRF by decoupling semantic or geometric selection from appearance rendering using a Distilled Feature Field (DFF) [S1]. It solves the problem of inconsistent multi-view edits by defining a selection probability $p$ for each 3D point based on external feature queries, allowing for precise operations like deletion, colorization, and geometric transformation that respect the underlying scene structure [S1]. It sets itself apart from standard optimization-based methods (e.g., [RePaint-NeRF](14365-repaint-nerf.md)) by avoiding iterative gradient descent for the edit itself, instead relying on a deterministic query-and-transform pipeline during rendering [S1].

## How it works
**Inputs:**
*   Pre-trained NeRF representation $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ [S1].
*   Pre-trained teacher network (e.g., LSeg or DINO) for feature extraction [S1].
*   User-defined editing parameters:
    *   Selection threshold $\tau$ (default not specified, user-defined) [S1].
    *   Colorization function $b(\cdot)$ [S1].
    *   Geometric transformation $g(\cdot)$ and its inverse $g^{-1}(\cdot)$ [S1].
*   Ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ for each pixel [S1].

**Outputs:**
*   Edited image $I'$ [S1].

**Data Structures:**
*   **DFF:** A field storing feature embeddings or probability scores $p$ for 3D coordinates. In the source, $p$ is calculated by querying the DFF for each point [S1]. The exact internal layout of the DFF is not specified, but it outputs a scalar probability $p \in [0, 1]$ representing the likelihood that coordinate $\mathbf{x}$ matches the editing query [S1].
*   **NeRF:** Standard MLP or grid-based representation storing density $\sigma$ and color $\mathbf{c}$ [S1].

**Steps:**
1.  **Ray Sampling:** Sample points $\{\mathbf{x}_i\}$ along the ray $\mathbf{r}$ as in standard volume rendering [S1].
2.  **Feature Query:** For each point $\mathbf{x}_i$, query the DFF to obtain the selection probability $p_i$ [S1].
3.  **Selection Decision:** Determine if $\mathbf{x}_i$ is "selected" by the edit query:
    *   If $p_i > \tau$, $\mathbf{x}_i$ is **selected**.
    *   If $p_i \le \tau$, $\mathbf{x}_i$ is **not selected**.
    *   *Note:* The source mentions an alternative mixture approach where results are blended in proportion to $p$, but the primary procedure uses the threshold [S1].
4.  **Transform Application:** For each point $\mathbf{x}_i$, compute the effective density $\sigma'_i$ and color $\mathbf{c}'_i$:
    *   **Case A: Not Selected:**
        *   Query the vanilla NeRF: $(\sigma_i, \mathbf{c}_i) = f_\theta(\mathbf{x}_i, \mathbf{d})$ [S1].
        *   Set $\sigma'_i = \sigma_i, \mathbf{c}'_i = \mathbf{c}_i$ [S1].
    *   **Case B: Selected:**
        *   **Deletion:** Set $\sigma'_i = 0$. Color is irrelevant as density is zero [S1].
        *   **Color Editing:** Query the vanilla NeRF for $(\sigma_i, \mathbf{c}_i)$. Apply colorization function $b$: $\mathbf{c}'_i = b(\mathbf{c}_i)$. Keep $\sigma'_i = \sigma_i$ [S1].
        *   **Translation / Rescaling:**
            1.  Compute deformed coordinate $\mathbf{x}_0 = g^{-1}(\mathbf{x}_i)$ [S1].
            2.  Query the DFF for $\mathbf{x}_0$ to get probability $p_0$.
            3.  If $\mathbf{x}_0$ is **selected** ($p_0 > \tau$):
                *   Query NeRF for $(\sigma_0, \mathbf{c}_0) = f_\theta(\mathbf{x}_0, \mathbf{d})$ [S1].
                *   Set $\sigma'_i = \sigma_0, \mathbf{c}'_i = \mathbf{c}_0$ [S1].
            4.  If $\mathbf{x}_0$ is **not selected**:
                *   The source does not explicitly define the fallback for the original point $\mathbf{x}_i$ in this specific branch, but implies the transformation logic applies primarily when the target location is also within the selected region. If $\mathbf{x}_0$ is not selected, standard behavior for "selected" points (likely querying original $\mathbf{x}_i$) is assumed [G].
            5.  **Boundary Mixing:** If both $\mathbf{x}_i$ and $\mathbf{x}_0$ are selected and have non-zero density (e.g., at object boundaries during deformation):
                *   Mix colors $\mathbf{c}_i$ and $\mathbf{c}_0$ in the ratio of their alphas $\alpha_i$ and $\alpha_0$ [S1].
                *   $\mathbf{c}'_i = \frac{\alpha_i \mathbf{c}_i + \alpha_0 \mathbf{c}_0}{\alpha_i + \alpha_0}$ [G] (Derived from "mix their colors... in the ratio of their alphas").
                *   Density handling for mixed points is not explicitly detailed beyond the color mixing instruction [S1].
5.  **Volume Rendering:** Perform standard volume rendering using the modified tuples $(\sigma'_i, \mathbf{c}'_i)$ along the ray [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray_o, ray_d, nerf_model, dff_model, transform_g_inv, color_func_b, threshold, is_deletion=False):
    """
    Renders a single ray with editing applied via DFF.
    """
    # 1. Sample points
    t_vals = sample_points_along_ray(ray_o, ray_d)
    points = ray_o + t_vals[:, None] * ray_d
    
    # 2. Query DFF for selection probabilities
    # dff_model returns a scalar probability p for each point
    probs = dff_model(points)
    
    # 3. Determine selection mask
    selected = probs > threshold
    
    densities = []
    colors = []
    
    for i, (x, p, is_sel) in enumerate(zip(points, probs, selected)):
        if not is_sel:
            # Case A: Not Selected - Use vanilla NeRF
            sigma, c = nerf_model(x, ray_d)
            densities.append(sigma)
            colors.append(c)
        else:
            # Case B: Selected - Apply Transform
            if is_deletion:
                # 4-A: Deletion
                densities.append(0.0)
                colors.append([0.0, 0.0, 0.0]) # Color irrelevant
            else:
                # Query original point for base values
                sigma_orig, c_orig = nerf_model(x, ray_d)
                
                # Check for geometric transformation (Translation/Rescaling)
                # Assume transform_g_inv is identity if not a geometric edit
                x_deformed = transform_g_inv(x)
                
                # Query DFF for deformed point
                p_deformed = dff_model(x_deformed)
                is_deformed_selected = p_deformed > threshold
                
                if is_deformed_selected:
                    # Get values from deformed location
                    sigma_def, c_def = nerf_model(x_deformed, ray_d)
                    
                    # 4'-C: Boundary Mixing
                    # Calculate alpha for mixing ratio
                    # Note: Alpha depends on density and step size. 
                    # For simplicity in this procedural step, we use density as proxy 
                    # or assume pre-computed alpha if available. 
                    # The source says "ratio of their alphas".
                    alpha_orig = 1.0 - torch.exp(-sigma_orig) # Approximation
                    alpha_def = 1.0 - torch.exp(-sigma_def)   # Approximation
                    
                    total_alpha = alpha_orig + alpha_def
                    if total_alpha > 0:
                        w_orig = alpha_orig / total_alpha
                        w_def = alpha_def / total_alpha
                        c_final = w_orig * c_orig + w_def * c_def
                    else:
                        c_final = c_orig
                        
                    densities.append(sigma_def) # Use density of deformed point
                    colors.append(c_final)
                else:
                    # Deformed point not selected. 
                    # Source does not specify fallback for original point in this branch.
                    # Assuming we fall back to original point's edited color if color edit,
                    # or original density if not deletion.
                    if color_func_b is not None:
                        c_final = color_func_b(c_orig)
                    else:
                        c_final = c_orig
                    
                    densities.append(sigma_orig)
                    colors.append(c_final)

    # 5. Volume Rendering
    # Standard alpha compositing
    accumulated_color = 0.0
    accumulated_alpha = 0.0
    T = 1.0
    
    for sigma, c in zip(densities, colors):
        alpha = 1.0 - torch.exp(-sigma)
        weight = T * alpha
        accumulated_color += weight * c
        accumulated_alpha += weight
        T *= (1.0 - alpha)
        
    return accumulated_color
```

## Subtopics

### [Local Editing via SAM and Depth Propagation](13674-local-editing-via-sam-and-depth-propagation.md)
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information. It solves the problem where existing methods induce simultaneous changes in both background and foreground, contradicting user intentions to alter distinct objects. It sets itself apart by using depth-based reprojection to ensure multi-view consistency of the segmentation mask, allowing exclusive modification of the target instance while leaving the background intact.

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views [S2]. The update formula is $I_i' = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w)$, where $m_w$ is the binary mask for the projected area and $\lambda$ controls the ratio preserved from the original pixel value [S2].

### NeRF-RPN Application
An application where scene editing is performed by setting the density inside a proposal bounding box (generated by a Region Proposal Network) to zero when rendering [S3]. This effectively removes objects detected by the framework [S3].

### NeRF-In
A simple 3D inpainting method that trains a NeRF using pixel reconstruction loss, guided by 2D inpainted images generated by an MST inpainting network [S4]. It relies on standard pixel loss rather than the DFF-based selection mechanism [S4].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the additional DFF query per sample point. If the DFF is a lightweight network or lookup table, the overhead is minimal compared to the NeRF query [S1].
*   **Memory Complexity:** Requires storing the DFF in addition to the NeRF. The size of the DFF depends on the feature dimension and spatial resolution, which is not specified in the sources [S1].
*   **Strengths:** Deterministic editing; no iterative optimization required for the edit itself; supports complex transformations (translation, colorization) [S1].
*   **Limitations:** Relies on the quality of the teacher network (LSeg/DINO) for accurate selection; threshold tuning required [S1].
*   **When to use:** When precise, non-iterative editing of specific semantic or geometric regions is required [S1].

## Background
The method originates from "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It utilizes pre-trained teacher networks like LSeg (ViT-L/16 image encoder, CLIP’s ViT-B/32 text encoder) and DINO to generate feature maps for distillation [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
[S4] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
