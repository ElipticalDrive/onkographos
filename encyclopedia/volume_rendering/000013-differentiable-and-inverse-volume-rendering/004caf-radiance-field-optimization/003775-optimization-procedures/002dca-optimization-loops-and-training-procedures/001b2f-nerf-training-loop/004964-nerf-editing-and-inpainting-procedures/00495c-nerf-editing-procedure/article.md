# NeRF Editing Procedure

Also called: DFF Editing Procedure, Feature-Field Distillation Editing

## Overview
The NeRF Editing Procedure enables semantic and geometric manipulation of a pre-trained Neural Radiance Field ($F_\Theta$) by intercepting the volume rendering pipeline with external guidance derived from 2D teacher networks or user-defined masks [S1]. Unlike standard training which optimizes $\Theta$ to fit input images, this method modifies the query logic during rendering to selectively apply transformations (deletion, colorization, translation) based on feature-space probabilities [S1]. It sets itself apart from direct optimization-based editing methods (like [DFF](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)'s training phase or [Blended-NeRF](12273-blended-nerf.md)'s blending) by performing edits as a post-hoc inference-time operation, allowing for immediate visual feedback without retraining the entire field [S1].

## How it works
The procedure operates on a pre-trained NeRF representation $F_\Theta$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$, augmented by a Distilled Feature Field (DFF) or similar feature encoder that provides semantic probabilities [S1].

### Inputs and Outputs
*   **Input**: Pre-trained NeRF parameters $\Theta$, a Distilled Feature Field (DFF) or teacher network features (e.g., from LSeg or DINO), a user-defined semantic query or mask, and an editing transformation $g$ (for translation/rescaling) or colorization function $b$ [S1].
*   **Output**: Edited rendered images or video frames.

### Data Structures
1.  **NeRF Model $F_\Theta$**: Standard MLP or grid-based representation.
    *   Input: $(\mathbf{x}, \mathbf{d}) \in \mathbb{R}^3 \times \mathbb{R}^3$.
    *   Output: $(\sigma, \mathbf{c}) \in \mathbb{R}^+ \times \mathbb{R}^3$.
2.  **Feature Encoder / DFF**: Maps 3D coordinates to semantic probability or feature vectors.
    *   Input: $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Probability $p \in [0, 1]$ indicating the likelihood that coordinate $\mathbf{x}$ belongs to the target semantic class defined by the query [S1].
3.  **Editing Parameters**:
    *   Threshold $\tau$: User-defined probability threshold for selection [S1].
    *   Transformation $g$: Geometric edit (e.g., translation vector).
    *   Colorization $b$: Function mapping color $\mathbf{c}$ to edited color [S1].

### Numbered Steps
1.  **Ray Sampling**: For each pixel, generate a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$. Sample $K$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_K\}$ along the ray using standard stratified or uniform sampling [S1].
2.  **Feature Query**: For each sample point $\mathbf{x}_k$, query the DFF to obtain the probability $p_k$ that $\mathbf{x}_k$ matches the semantic query [S1].
3.  **Selection Decision**: Define a binary selection state $s_k$:
    $$ s_k = \begin{cases} 1 & \text{if } p_k \geq \tau \\ 0 & \text{otherwise} \end{cases} $$
    Alternatively, use the probability $p_k$ as a soft weight for mixing [S1].
4.  **Point Transformation & Query**:
    *   If the editing involves geometric transformation (translation/rescaling) defined by $g$:
        *   Compute the inverse-transformed coordinate $\mathbf{x}'_k = g^{-1}(\mathbf{x}_k)$ [S1].
        *   Query the DFF for $\mathbf{x}'_k$ to get probability $p'_k$.
        *   Determine selection $s'_k$ for $\mathbf{x}'_k$ using threshold $\tau$ [S1].
5.  **Density and Color Calculation**: For each point $\mathbf{x}_k$, determine the effective density $\tilde{\sigma}_k$ and color $\tilde{\mathbf{c}}_k$:
    *   **Case A: Not Selected ($s_k = 0$)**:
        *   Query vanilla NeRF: $(\sigma_k, \mathbf{c}_k) = F_\Theta(\mathbf{x}_k, \mathbf{d})$ [S1].
        *   $\tilde{\sigma}_k = \sigma_k$, $\tilde{\mathbf{c}}_k = \mathbf{c}_k$.
    *   **Case B: Selected ($s_k = 1$)**:
        *   **Deletion**: Set $\tilde{\sigma}_k = 0$ [S1].
        *   **Color Editing**: Query vanilla NeRF for $(\sigma_k, \mathbf{c}_k)$. Apply colorization: $\tilde{\mathbf{c}}_k = b(\mathbf{c}_k)$. Density remains $\tilde{\sigma}_k = \sigma_k$ [S1].
        *   **Translation/Rescaling**:
            *   If $\mathbf{x}'_k$ is also selected ($s'_k = 1$):
                *   Query vanilla NeRF for $(\sigma'_k, \mathbf{c}'_k) = F_\Theta(\mathbf{x}'_k, \mathbf{d})$ [S1].
                *   Compute alphas: $\alpha_k = 1 - e^{-\sigma_k \delta_k}$ and $\alpha'_k = 1 - e^{-\sigma'_k \delta_k}$ (where $\delta_k$ is the step size) [S1].
                *   Mix colors based on alpha ratio:
                    $$ \tilde{\mathbf{c}}_k = \frac{\alpha_k \mathbf{c}_k + \alpha'_k \mathbf{c}'_k}{\alpha_k + \alpha'_k} $$
                *   Set density $\tilde{\sigma}_k$ to the sum or appropriate combination to maintain volume integrity (simplified as using the transformed point's properties if dominant, or mixing densities similarly) [S1]. *Note: The source specifies mixing colors in the ratio of their alphas for simplicity; density handling at boundaries is implied to prevent holes.*
            *   If $\mathbf{x}'_k$ is not selected ($s'_k = 0$):
                *   Use original $\mathbf{x}_k$ properties: $\tilde{\sigma}_k = \sigma_k$, $\tilde{\mathbf{c}}_k = \mathbf{c}_k$ (or apply color edit if specified) [S1].
6.  **Volume Rendering**: Perform standard volumetric rendering using the modified tuples $(\tilde{\sigma}_k, \tilde{\mathbf{c}}_k)$:
    $$ C(\mathbf{d}) = \sum_{k=1}^K T_k \tilde{\mathbf{c}}_k (1 - e^{-\tilde{\sigma}_k \delta_k}), \quad T_k = e^{-\sum_{j=1}^{k-1} \tilde{\sigma}_j \delta_j} $$
    [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray_origin, ray_direction, nerf_model, dff_model, 
                      editing_type, transformation_g, colorization_b, 
                      threshold_tau, num_samples, near, far):
    """
    Renders a single ray with NeRF editing applied.
    
    Args:
        ray_origin: 3D vector, camera origin.
        ray_direction: 3D vector, normalized ray direction.
        nerf_model: Function F_Theta(x, d) -> (sigma, color).
        dff_model: Function P(x) -> probability p.
        editing_type: 'deletion', 'color', 'translation'.
        transformation_g: Function g^-1(x) for inverse transform.
        colorization_b: Function b(c) for color edit.
        threshold_tau: Float, probability threshold for selection.
        num_samples: Int, number of points per ray.
        near, far: Floats, ray clipping bounds.
        
    Returns:
        color: 3D vector, final pixel color.
    """
    # 1. Sample points
    t_vals = np.linspace(near, far, num_samples)
    points = ray_origin + t_vals[:, None] * ray_direction
    
    # 2. Query DFF for original points
    probs_orig = dff_model(points)
    selected_orig = probs_orig >= threshold_tau
    
    # 3. Handle Geometric Transformation if applicable
    if editing_type == 'translation':
        # Compute inverse transformed coordinates
        points_inv = transformation_g(points) # g^-1(x)
        probs_inv = dff_model(points_inv)
        selected_inv = probs_inv >= threshold_tau
    else:
        points_inv = None
        selected_inv = None

    # 4. Calculate modified density and color for each point
    final_colors = np.zeros_like(points)
    final_sigmas = np.zeros(num_samples)
    
    # Query vanilla NeRF for all points (optimization: batch query)
    sigmas_orig, colors_orig = nerf_model(points, ray_direction)
    
    if editing_type == 'translation':
        sigmas_inv, colors_inv = nerf_model(points_inv, ray_direction)
        
    for k in range(num_samples):
        if not selected_orig[k]:
            # Not selected: use vanilla NeRF
            final_sigmas[k] = sigmas_orig[k]
            final_colors[k] = colors_orig[k]
        else:
            # Selected
            if editing_type == 'deletion':
                final_sigmas[k] = 0.0
                final_colors[k] = colors_orig[k] # Color irrelevant if sigma=0
            elif editing_type == 'color':
                final_sigmas[k] = sigmas_orig[k]
                final_colors[k] = colorization_b(colors_orig[k])
            elif editing_type == 'translation':
                if selected_inv[k]:
                    # Both original and transformed points are selected
                    # Mix colors based on alpha ratio
                    # Alpha = 1 - exp(-sigma * delta)
                    # Assuming uniform delta for simplicity or passing delta
                    delta = (far - near) / num_samples
                    alpha_orig = 1.0 - np.exp(-sigmas_orig[k] * delta)
                    alpha_inv = 1.0 - np.exp(-sigmas_inv[k] * delta)
                    
                    # Mix colors
                    total_alpha = alpha_orig + alpha_inv
                    if total_alpha > 0:
                        final_colors[k] = (alpha_orig * colors_orig[k] + 
                                           alpha_inv * colors_inv[k]) / total_alpha
                    else:
                        final_colors[k] = colors_orig[k]
                    
                    # Density handling: Source implies mixing for simplicity.
                    # A robust implementation might sum densities or use max.
                    # Here we keep original sigma for transmittance calculation 
                    # but note that the visual contribution is mixed.
                    final_sigmas[k] = sigmas_orig[k] 
                else:
                    # Only original is selected, transformed is not
                    # Use original properties (or apply color edit if combined)
                    final_sigmas[k] = sigmas_orig[k]
                    final_colors[k] = colors_orig[k]
    
    # 5. Volume Rendering
    # Calculate transmittance
    deltas = (far - near) / num_samples
    alpha = 1.0 - np.exp(-final_sigmas * deltas)
    T = np.exp(-np.cumsum(final_sigmas * deltas))
    T = np.concatenate(([1.0], T[:-1])) # T_0 = 1
    
    # Composite color
    contribution = T * alpha[:, None] * final_colors
    color = np.sum(contribution, axis=0)
    
    return color
```

### Parameters and Edge Cases
*   **Threshold $\tau$**: Controls the strictness of semantic selection. A higher $\tau$ results in more conservative editing. Not given in the sources.
*   **Precision**: Floating-point precision in alpha mixing is critical at boundaries to avoid artifacts [S1].
*   **Degenerate Input**: If $g^{-1}(\mathbf{x})$ maps to a point outside the bounding box of the NeRF, the NeRF query should return zero density or be clamped [G].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive masks for all components in each image, allowing users to isolate a target instance with a single point [S2]. The identified mask is projected across different views using depth information, continuing as long as the projected mask sufficiently overlaps with the corresponding mask in the current view [S2]. This integrated multi-view mask permits exclusive modification of the target object, leaving the background intact [S2]. A key pitfall of existing methods is inducing simultaneous changes in both background and foreground, which this method aims to avoid [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire iterative editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views [S2]. The update for a given view $i$ from an edited view $j$ is calculated as:
$$ I'_i = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w) $$
where $m_w$ is the binary mask for the projected area and $\lambda$ is a hyperparameter controlling the ratio preserved from the original pixel value [S2]. This accelerates the process by providing a consistent initial state for subsequent iterative refinement [S2].

## Operating characteristics
*   **Time Complexity**: The editing procedure itself is $O(K)$ per ray, where $K$ is the number of samples, similar to standard NeRF rendering. However, it requires an additional query to the DFF/teacher network for each point, increasing constant factors [S1]. The warm-up strategy in ViCA-NeRF adds an initial projection cost but reduces the number of iterative optimization steps required [S2].
*   **Memory Complexity**: Requires storing the DFF or teacher network features, which can be significant if high-resolution feature maps are used [S1].
*   **Strengths**: Allows for immediate, non-destructive editing without retraining the NeRF [S1]. Supports semantic, geometric, and color edits simultaneously [S1].
*   **Limitations**: Quality depends heavily on the accuracy of the DFF/teacher network features [S1]. Geometric edits may suffer from artifacts at boundaries if the inverse transformation maps points to untrained regions [S1].
*   **When to use**: When rapid prototyping of edits is needed, or when the editing semantics are well-captured by 2D teacher networks.
*   **When not to use**: When high-fidelity geometric consistency is required without the benefit of iterative optimization, or when the teacher network fails to segment the target object accurately.

## Background
The NeRF Editing Procedure was introduced in the context of Decomposing NeRF for Editing Via Feature Field Distillation (DFF) [S1]. It leverages pre-trained 2D vision models (like LSeg and DINO) to distill semantic information into a 3D feature field, enabling editing based on semantic queries rather than just pixel colors [S1]. The method builds on the standard NeRF volume rendering pipeline but modifies the query logic to incorporate these features [S1]. Variants like ViCA-NeRF extend this by adding view-consistency-aware warm-up and local editing strategies using SAM and depth propagation [S2].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
