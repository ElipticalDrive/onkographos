# NeRF Editing Procedure

Also called: Rendering-time Transformation, Feature-Field Distillation Editing, DFF Editing

## Overview
The NeRF Editing Procedure modifies a radiance field representation by altering density $\sigma$ and color $\mathbf{c}$ values on-the-fly during volume rendering, based on semantic probabilities or geometric masks, without changing the stored network weights [S1, 12491-nerf-editing-procedure.md]. It solves the problem of consistent 3D editing (deletion, colorization, translation) by leveraging a pre-trained teacher network (e.g., LSeg, DINO) to provide semantic guidance, distinguishing it from optimization-based methods that require iterative gradient updates [S1, 12372-nerf-in-inpainting-optimization.md].

## How it works
**Inputs:**
*   A trained vanilla NeRF model $\mathcal{N}$ with parameters $\Theta$, mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
*   A Feature Field Distillation (DFF) model $\mathcal{F}$ (e.g., LSeg or DINO) providing semantic probabilities.
*   A ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ for each pixel.
*   Editing parameters: transformation function $g$ (and its inverse $g^{-1}$), colorization function $b$, and selection threshold $\tau$.

**Outputs:**
*   Rendered image $\mathbf{I}$ with edited regions.

**Data Structures:**
*   **DFF Output:** For a point $\mathbf{x}$, the DFF returns a probability vector or scalar $p \in [0, 1]$ indicating the likelihood that $\mathbf{x}$ belongs to the target semantic class defined by the query.
*   **NeRF Output:** $\sigma(\mathbf{x}) \in \mathbb{R}_{\ge 0}$ (density) and $\mathbf{c}(\mathbf{x}) \in [0, 1]^3$ (color).

**Steps:**

1.  **Ray Sampling:** Sample points $\{\mathbf{x}_1, \dots, \mathbf{x}_K\}$ along the ray $\mathbf{r}$ at depths $t_1, \dots, t_K$ as in standard NeRF [S1].
2.  **Semantic Query:** For each point $\mathbf{x}_k$, query the DFF to calculate the probability $p_k$ that the coordinate matches the user-defined semantic query [S1].
3.  **Selection Decision:** Define the point as "selected" if $p_k > \tau$ (user-defined threshold), otherwise "not selected". Alternatively, use a soft mixture proportional to $p_k$ [S1].
4.  **Transform Application:**
    *   **If not selected:** Compute standard density $\sigma_k = \sigma(\mathbf{x}_k)$ and color $\mathbf{c}_k = \mathbf{c}(\mathbf{x}_k)$ using the vanilla NeRF [S1].
    *   **If selected:** Apply one of the following transforms [S1]:
        *   **Deletion:** Set density $\sigma_k = 0$. Color $\mathbf{c}_k$ is irrelevant as the point becomes transparent.
        *   **Color Editing:** Query the NeRF for $\sigma_k$ and $\mathbf{c}_k$. Transform the color using a colorization function $b$: $\mathbf{c}_k' = b(\mathbf{c}_k)$. Use $\sigma_k$ and $\mathbf{c}_k'$ for rendering.
        *   **Translation/Rescaling:**
            1.  Compute the deformed coordinate $\mathbf{x}_0 = g^{-1}(\mathbf{x}_k)$, where $g$ is the editing transformation (e.g., translation vector addition) [S1].
            2.  Query the DFF for $\mathbf{x}_0$ to get probability $p_0$.
            3.  If $\mathbf{x}_0$ is selected ($p_0 > \tau$), use $\mathbf{x}_0$ to query the NeRF for $\sigma_0$ and $\mathbf{c}_0$.
            4.  If both $\mathbf{x}_k$ and $\mathbf{x}_0$ are selected and have non-zero density (e.g., at boundaries), mix their colors $\mathbf{c}_k$ and $\mathbf{c}_0$ in the ratio of their alphas $\alpha_k$ and $\alpha_0$ at the point for simplicity [S1].
            5.  Use the resulting $\sigma$ and $\mathbf{c}$ for rendering.
5.  **Volume Rendering:** Perform standard volume rendering using the series of $(\sigma_k, \mathbf{c}_k')$ tuples to compute the final pixel color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray_origin, ray_direction, nerf_model, dff_model, 
                      transform_func, colorize_func, threshold, alpha_composite):
    """
    Renders a single ray with editing applied at rendering time.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        nerf_model: Function mapping (x, d) -> (sigma, c).
        dff_model: Function mapping x -> probability p.
        transform_func: Function g(x) -> x_transformed.
        colorize_func: Function c -> c_edited.
        threshold: Float, semantic selection threshold.
        alpha_composite: Function to blend colors based on alpha.
        
    Returns:
        final_color: 3D vector, the edited color for the pixel.
    """
    # Standard NeRF sampling parameters
    num_samples = 128
    near = 0.0
    far = 10.0
    t_vals = np.linspace(near, far, num_samples)
    
    # Sample points along the ray
    points = ray_origin + ray_direction * t_vals[:, np.newaxis]
    
    # Pre-calculate semantic probabilities for all points
    # Note: In practice, DFF queries might be batched or cached
    probs = dff_model.query_semantic(points)
    
    selected_mask = probs > threshold
    
    densities = np.zeros(num_samples)
    colors = np.zeros((num_samples, 3))
    
    for i in range(num_samples):
        x = points[i]
        p = probs[i]
        
        if not selected_mask[i]:
            # Standard rendering for unselected regions
            sigma, c = nerf_model(x, ray_direction)
            densities[i] = sigma
            colors[i] = c
        else:
            # Editing logic for selected regions
            # Check if geometric transform is needed (e.g., translation)
            # If transform_func is identity, x0 = x
            x0 = transform_func(x)
            
            # Query semantic for transformed point if it differs
            if not np.allclose(x, x0):
                p0 = dff_model.query_semantic(x0)
                selected_x0 = p0 > threshold
            else:
                selected_x0 = selected_mask[i]
                
            # Get NeRF outputs for original point
            sigma_orig, c_orig = nerf_model(x, ray_direction)
            
            # Get NeRF outputs for transformed point if selected
            if selected_x0:
                sigma_trans, c_trans = nerf_model(x0, ray_direction)
            else:
                sigma_trans = 0.0
                c_trans = c_orig # Fallback, though density 0 makes it transparent
            
            # Apply specific edit type
            # Assuming 'translation/rescaling' logic for mixed case, 
            # or simple colorization if no geometric transform.
            # For simplicity, we assume colorization if no geometric shift,
            # or geometric blending if shifted.
            
            if np.allclose(x, x0):
                # Color Editing
                c_final = colorize_func(c_orig)
                densities[i] = sigma_orig
                colors[i] = c_final
            else:
                # Translation/Rescaling with boundary mixing
                # Calculate alpha for each component at this point
                # Alpha is derived from density and step size dt
                dt = t_vals[1] - t_vals[0] # Approximate step size
                
                # Alpha contribution of original point
                alpha_orig = 1.0 - np.exp(-sigma_orig * dt)
                # Alpha contribution of transformed point
                alpha_trans = 1.0 - np.exp(-sigma_trans * dt)
                
                # Mix colors based on alpha ratio if both are non-zero
                if sigma_orig > 0 and sigma_trans > 0:
                    # Simple alpha-weighted mix as per source [S1]
                    total_alpha = alpha_orig + alpha_trans
                    if total_alpha > 0:
                        w_orig = alpha_orig / total_alpha
                        w_trans = alpha_trans / total_alpha
                        c_final = w_orig * c_orig + w_trans * c_trans
                    else:
                        c_final = c_orig
                else:
                    # If only one has density, use that color
                    if sigma_orig > 0:
                        c_final = c_orig
                    else:
                        c_final = c_trans
                        
                # Density is the sum (approximation for blending) or max?
                # Source says "mix colors... in ratio of their alphas".
                # Density for volume rendering usually accumulates.
                # Here we treat them as competing contributions.
                # We set density to the one that "wins" or sum?
                # Standard volume rendering sums densities. 
                # However, if we are replacing the object, we might want to 
                # use the density of the transformed object if it's the primary edit.
                # The source implies using the transformed point's properties 
                # if selected. Let's use the transformed density if selected, 
                # otherwise original.
                if selected_x0:
                    densities[i] = sigma_trans
                else:
                    densities[i] = sigma_orig
                colors[i] = c_final

    # Standard Volume Rendering
    alphas = 1.0 - np.exp(-densities * (t_vals[1:] - t_vals[:-1]))
    transmittance = np.exp(-np.cumsum(densities * (t_vals[1:] - t_vals[:-1])))
    
    # Add the last segment
    alphas = np.concatenate([alphas, [1.0]])
    transmittance = np.concatenate([[1.0], transmittance])
    
    weights = alphas * transmittance
    final_color = np.sum(weights[:, np.newaxis] * colors, axis=0)
    
    return final_color
```

**Parameters:**
*   $\tau$ (threshold): User-defined semantic probability threshold. Typical values not given in sources [S1].
*   $b(\cdot)$: Colorization function. Implementation depends on the specific colorization model (e.g., a small MLP or lookup table) [S1].
*   $g(\cdot)$: Geometric transformation function. For translation, $g(\mathbf{x}) = \mathbf{x} + \mathbf{v}$ [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Mixing:** At the boundary between edited and unedited regions, or between the original and transformed object, both $\mathbf{x}$ and $\mathbf{x}_0$ may be selected. The source specifies mixing colors in the ratio of their alphas for simplicity [S1].
*   **DFF Accuracy:** The quality of editing depends heavily on the semantic accuracy of the DFF. Incorrect probabilities lead to incorrect selection [S1].
*   **Transformation Consistency:** For translation/rescaling, the inverse transformation $g^{-1}$ must be correctly applied to map the rendered point back to the source object's coordinate space [S1].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive masks, allows users to isolate a target instance with a single point, and projects the mask across views using depth until overlap is sufficient. This ensures multi-view consistent segmentation for exclusive modification of the target object [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views. The update for a view $i$ from an edited view $j$ is calculated as $I_i' = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w)$, where $m_w$ is the binary mask for the projected area and $\lambda$ controls the ratio preserved from the original pixel value [S2].

### NeRF-RPN Application
A method to edit the scene by setting the density inside a proposal bounding box to zero when rendering, effectively deleting the object detected by the NeRF-RPN framework [S3].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the NeRF query and the DFF query per sample point. Since no iterative optimization is required during rendering, it is significantly faster than optimization-based editing methods [S1, S2].
*   **Memory Complexity:** Requires storing the NeRF weights and the DFF model weights. No additional per-scene optimization buffers are needed [S1].
*   **Strengths:** Real-time or near-real-time editing capability; preserves the integrity of unedited regions by design; leverages powerful pre-trained semantic models [S1, S2].
*   **Limitations:** Editing quality is constrained by the semantic accuracy of the DFF; geometric edits (translation) may cause artifacts at boundaries if the transformation is not perfectly aligned with the object's geometry [S1, S2].
*   **When to use:** When real-time editing is required, or when the scene structure is well-captured by the pre-trained semantic features [S1, S2].
*   **When not to use:** When high-fidelity geometric changes are required that cannot be captured by simple transformations, or when the semantic classes are not covered by the DFF [S1].

## Background
The NeRF Editing Procedure was introduced in the context of Decomposing NeRF for Editing Via Feature Field Distillation (DFF) [S1]. It builds on the observation that pre-trained teacher networks like LSeg and DINO can provide semantic guidance for NeRF editing without requiring retraining of the NeRF itself [S1]. The method was developed to address the inefficiency of iterative optimization-based editing methods [S2].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
