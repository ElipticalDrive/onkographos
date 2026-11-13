# NeRF Editing Procedure

Also called: Feature Field Distillation Editing, DFF-based Editing

## Overview
The NeRF Editing Procedure modifies the standard volume rendering process by integrating semantic or geometric guidance fields (such as Distillation Feature Fields) to selectively alter density and color at specific 3D coordinates [S1]. Unlike global optimization methods that retrain the entire network, this approach intercepts the ray-marching loop to apply transforms (deletion, colorization, or geometric warping) only where a query field indicates a match [S1]. It sets apart from sibling methods like [Blended-NeRF](12273-blended-nerf.md) by operating directly on the feature query probability rather than blending rendered images, and from [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md) by using explicit coordinate selection rather than pixel-space guidance [S1, 12372-nerf-in-inpainting-optimization.md].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, a Distillation Feature Field (DFF) $D$ that outputs a probability $p$ of coordinate matching a query, a camera ray $\mathbf{r}$, and an editing operation type (Deletion, Color, Translation/Rescaling) [S1].
*   **Output:** A rendered color $\hat{C}(\mathbf{r})$ for the ray [S1].

**Data Structures:**
*   **Ray Samples:** A list of $N$ points $\mathbf{x}_i$ along the ray, with depths $t_i$.
*   **DFF Query:** A function returning a scalar probability $p \in [0, 1]$ for a given 3D coordinate $\mathbf{x}$ [S1].
*   **Threshold:** A user-defined scalar $\tau$ for coordinate selection [S1].

**Numbered Steps:**
1.  **Sample Points:** Sample $N$ points $\mathbf{x}_i$ along the ray $\mathbf{r}$ as in standard NeRF [S1].
2.  **Query DFF:** For each point $\mathbf{x}_i$, query the DFF to obtain the probability $p_i$ that $\mathbf{x}_i$ matches the editing query [S1].
3.  **Select Coordinates:** Define a coordinate $\mathbf{x}_i$ as "selected" if $p_i > \tau$. Otherwise, it is "not selected" [S1].
    *   *Note:* The source also mentions an alternative where selected and not-selected results are mixed in proportion to $p_i$ without a hard threshold [S1].
4.  **Compute Properties:** For each point $\mathbf{x}_i$, determine the density $\sigma_i$ and color $\mathbf{c}_i$ based on the selection status and editing type:
    *   **If not selected:** Compute $\sigma_i = f_\theta(\mathbf{x}_i, \mathbf{d}).\sigma$ and $\mathbf{c}_i = f_\theta(\mathbf{x}_i, \mathbf{d}).\mathbf{c}$ using the vanilla NeRF [S1].
    *   **If selected:** Apply the specific transform (see below).
5.  **Volume Rendering:** Perform standard volume rendering using the resulting series of $(\sigma_i, \mathbf{c}_i)$ tuples [S1].

**Editing Transforms (for selected points):**
*   **Deletion:** Set density $\sigma_i = 0$. Color $\mathbf{c}_i$ is irrelevant as the term $(1 - \exp(-\sigma_i \delta_i))$ becomes 0 [S1].
*   **Color Editing:** Query the NeRF for $\sigma_i$ and $\mathbf{c}_i$. Transform the color using a colorization function $b$: $\mathbf{c}_i \leftarrow b(\mathbf{c}_i)$ [S1].
*   **Translation / Rescaling:**
    1.  Compute the inverse editing transformation $g^{-1}$ (e.g., inverse translation vector).
    2.  Calculate the deformed coordinate $\mathbf{x}_0 = g^{-1}(\mathbf{x}_i)$ [S1].
    3.  Query the DFF for $\mathbf{x}_0$ to get probability $p_0$.
    4.  **If $\mathbf{x}_0$ is selected:** Use $\mathbf{x}_0$ instead of $\mathbf{x}_i$ to query the NeRF for $\sigma_0$ and $\mathbf{c}_0$. Set $\sigma_i = \sigma_0, \mathbf{c}_i = \mathbf{c}_0$ [S1].
    5.  **If both $\mathbf{x}_i$ and $\mathbf{x}_0$ are selected and have non-zero density:** Mix the colors $\mathbf{c}_i$ and $\mathbf{c}_0$ in the ratio of their alphas (transmittance contributions) at the point for simplicity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray, nerf_model, dff_query, threshold, edit_type, color_func, inverse_transform):
    """
    Renders a single ray with NeRF editing.
    :param ray: Ray origin o and direction d.
    :param nerf_model: Pre-trained NeRF function f_theta.
    :param dff_query: Function D(x) returning probability p.
    :param threshold: Float tau for coordinate selection.
    :param edit_type: String 'deletion', 'color', or 'translation'.
    :param color_func: Function b for color editing.
    :param inverse_transform: Function g_inv for translation/rescaling.
    :return: Rendered color C.
    """
    # 1. Sample points
    t_vals = sample_points_along_ray(ray) # Standard NeRF sampling
    points = [ray.o + t * ray.d for t in t_vals]
    
    accumulated_transmittance = 1.0
    accumulated_color = [0.0, 0.0, 0.0]
    
    for i, x in enumerate(points):
        # 2. Query DFF
        p = dff_query(x)
        
        # 3. Select coordinates
        is_selected = p > threshold
        
        sigma = 0.0
        color = [0.0, 0.0, 0.0]
        
        if not is_selected:
            # 4a. Vanilla NeRF
            sigma, color = nerf_model(x, ray.d)
        else:
            # 4b. Apply transforms
            if edit_type == 'deletion':
                sigma = 0.0
                color = [0.0, 0.0, 0.0] # Irrelevant
            elif edit_type == 'color':
                sigma, original_color = nerf_model(x, ray.d)
                color = color_func(original_color)
            elif edit_type == 'translation':
                # Compute deformed coordinate
                x0 = inverse_transform(x)
                p0 = dff_query(x0)
                is_selected_x0 = p0 > threshold
                
                if is_selected_x0:
                    sigma0, color0 = nerf_model(x0, ray.d)
                    
                    # Query original point for mixing if needed
                    sigma_orig, color_orig = nerf_model(x, ray.d)
                    
                    # Check if both have non-zero density (boundary case)
                    if sigma0 > 0 and sigma_orig > 0:
                        # Mix colors in ratio of their alphas
                        # Alpha at point i is approx (1 - exp(-sigma * delta))
                        # For simplicity in mixing ratio, use sigma directly as proxy for alpha contribution
                        # Note: Source says "ratio of their alphas at the point"
                        alpha0 = 1.0 - math.exp(-sigma0 * delta_t)
                        alpha_orig = 1.0 - math.exp(-sigma_orig * delta_t)
                        
                        total_alpha = alpha0 + alpha_orig
                        if total_alpha > 0:
                            w0 = alpha0 / total_alpha
                            w_orig = alpha_orig / total_alpha
                            color = [w0 * c0 + w_orig * c_orig for c0, c_orig in zip(color0, color_orig)]
                        else:
                            color = color0
                        sigma = sigma0 # Use density of transformed point? Source implies using x0 for calc.
                        # Source: "x0 is used instead of x for calculating color and density"
                        # But then mentions mixing. We assume sigma is also mixed or taken from x0.
                        # Given "mix their colors... for simplicity", we assume sigma is dominated by the transformed geometry.
                        sigma = sigma0 
                    else:
                        sigma = sigma0
                        color = color0
                else:
                    # x0 not selected, fall back to x? Source implies x0 used *if* selected.
                    # If x0 not selected, we likely treat x as not selected or use x's properties.
                    # However, x was selected. This edge case is not fully detailed.
                    # Assuming fallback to vanilla x properties if x0 invalid.
                    sigma, color = nerf_model(x, ray.d)
        
        # 5. Volume Rendering Step
        # T_i = accumulated_transmittance
        # weight = T_i * (1 - exp(-sigma * delta_t))
        # accumulated_color += weight * color
        # accumulated_transmittance *= exp(-sigma * delta_t)
        
        delta_t = t_vals[i+1] - t_vals[i] if i < len(t_vals)-1 else 0.0 # Approximation
        # Note: Standard NeRF uses pre-computed deltas. Here we assume delta_t is available.
        
        transmittance_contribution = 1.0 - math.exp(-sigma * delta_t)
        weight = accumulated_transmittance * transmittance_contribution
        
        accumulated_color = [c + weight * col for c, col in zip(accumulated_color, color)]
        accumulated_transmittance *= math.exp(-sigma * delta_t)
        
    return accumulated_color
```

**Parameters:**
*   **Threshold ($\tau$):** User-defined probability threshold for coordinate selection. Typical values not given in sources [S1].
*   **Colorization Function ($b$):** Depends on the specific editing task (e.g., style transfer model) [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Mixing:** When both original and transformed coordinates are selected and have non-zero density (e.g., object boundaries), colors are mixed based on alpha ratios [S1].
*   **Threshold Sensitivity:** The choice of $\tau$ determines the precision of the edit. A mixture approach (proportional to $p$) is an alternative to hard thresholding [S1].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive masks, allows users to pinpoint targets with a single point, and projects masks across views using depth until sufficient overlap is achieved [S2]. This addresses the pitfall of existing methods changing both background and foreground simultaneously [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views using the formula:
$$
I_i' = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w)
$$
where $m_w$ is the binary mask for the projected area and $\lambda$ controls the ratio preserved from the original pixel value [S2].

### NeRF-RPN Application
An application where the scene is edited by setting the density inside a proposal bounding box (from a Region Proposal Network) to zero when rendering [S3].

## Operating characteristics
*   **Time Complexity:** Depends on the cost of querying the DFF and the NeRF for each sample point. No explicit asymptotic complexity is given in the sources [S1].
*   **Memory Complexity:** Requires storing the NeRF parameters and the DFF parameters. No specific memory footprint is given [S1].
*   **Strengths:** Allows for selective editing (deletion, colorization, translation) without retraining the entire NeRF from scratch [S1]. Can be combined with post-refinement strategies for higher consistency [S2].
*   **Limitations:** Relies on the quality of the DFF and the accuracy of depth information for propagation methods [S1, S2]. Incorrect depth estimation can affect results, though the model can manage some inaccuracies if the NeRF is built successfully [S2].
*   **When to use:** When specific objects or regions need to be modified (e.g., deletion, color change) while preserving the rest of the scene, and a suitable feature field or segmentation mask is available [S1, S2].

## Background
The NeRF Editing Procedure was introduced in "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It leverages Distillation Feature Fields (DFF) to decompose the NeRF representation for editing tasks. The method builds on the standard NeRF rendering equation but modifies the sample properties based on semantic queries [S1]. Variants like ViCA-NeRF [S2] and NeRF-RPN [S3] extend these concepts to view-consistency-aware editing and object-detection-based editing, respectively.

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
