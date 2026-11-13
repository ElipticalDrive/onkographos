# NeRF Editing Procedure

Also called: Inference-time NeRF Editing, Feature-Field Editing, Interception-based Editing

## Overview
The NeRF Editing Procedure modifies the appearance or geometry of a pre-trained Neural Radiance Field (NeRF) $F_\Theta$ without retraining the network parameters $\Theta$. It operates by intercepting the volume rendering process at inference time, querying an external feature field or segmentation model to identify regions of interest (ROI), and applying transformations to the density $\sigma$ and color $\mathbf{c}$ values of sampled points within those regions [S1, S2, S3]. Unlike optimization-based methods that update $\Theta$ via gradient descent, this approach modifies the rendering equation directly, allowing for real-time or near-real-time editing [S1]. It sets itself apart from siblings like [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md) by avoiding iterative backpropagation, and from [Blended-NeRF](12273-blended-nerf.md) by relying on semantic/feature priors rather than just geometric blending [S1, S2].

## How it works
**Inputs:**
*   Pre-trained NeRF representation $F_\Theta: (\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$ [S1].
*   External guidance source: Feature Field (e.g., DFF derived from LSeg/DINO) [S1], 2D segmentation masks (e.g., SAM) [S2], or 3D bounding box proposals [S3].
*   Editing parameters: Threshold $T$ for selection, transformation function $g$ (for translation/rescaling), colorization function $b$ [S1].

**Outputs:**
*   Edited rendered image $I_{edit}$.

**Data Structures:**
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{v}$.
*   **Sampled Points:** $\{\mathbf{x}_1, \mathbf{x}_2, \dots, \mathbf{x}_N\}$ along the ray, where $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{v}$ [S1].
*   **Feature Probability:** $p(\mathbf{x}) \in [0, 1]$, representing the likelihood that point $\mathbf{x}$ belongs to the target semantic class or object [S1].

**Numbered Steps:**

1.  **Ray Casting and Sampling:** For each pixel in the target view, cast a ray and sample $N$ points $\{\mathbf{x}_i\}_{i=1}^N$ along the ray, typically using uniform or importance sampling as in standard NeRF [S1].
2.  **Feature Query and Selection:** For each point $\mathbf{x}_i$, query the external guidance source to obtain the selection probability $p_i$ [S1].
    *   Define a binary selection mask $m_i$:
        $$ m_i = \begin{cases} 1 & \text{if } p_i \ge T \\ 0 & \text{otherwise} \end{cases} $$
    *   Alternatively, use the probability $p_i$ directly as a soft weight [S1].
3.  **Point Transformation (Optional):** If the edit involves geometric transformation (translation/rescaling) defined by $g$:
    *   Compute the inverse coordinate $\mathbf{x}_i' = g^{-1}(\mathbf{x}_i)$ [S1].
    *   Query the guidance source for $\mathbf{x}_i'$ to get $p_i'$ [S1].
4.  **Density and Color Modification:** For each point $\mathbf{x}_i$, determine the effective density $\tilde{\sigma}_i$ and color $\tilde{\mathbf{c}}_i$:
    *   **Case 1: Not Selected ($m_i = 0$):**
        *   Query the NeRF: $(\sigma_i, \mathbf{c}_i) = F_\Theta(\mathbf{x}_i, \mathbf{v})$ [S1].
        *   $\tilde{\sigma}_i = \sigma_i, \quad \tilde{\mathbf{c}}_i = \mathbf{c}_i$.
    *   **Case 2: Selected ($m_i = 1$):**
        *   **Deletion:** Set $\tilde{\sigma}_i = 0$ [S1, S3].
        *   **Color Editing:** Query NeRF for $(\sigma_i, \mathbf{c}_i)$. Apply colorization: $\tilde{\mathbf{c}}_i = b(\mathbf{c}_i)$ [S1].
        *   **Translation/Rescaling:**
            *   If $\mathbf{x}_i'$ is also selected ($m_i' = 1$):
                *   Query NeRF for $(\sigma_i, \mathbf{c}_i)$ at $\mathbf{x}_i$ and $(\sigma_i', \mathbf{c}_i')$ at $\mathbf{x}_i'$ [S1].
                *   Compute alpha values $\alpha_i = 1 - e^{-\sigma_i \delta_i}$ and $\alpha_i' = 1 - e^{-\sigma_i' \delta_i}$ (assuming equal step size $\delta$) [S1].
                *   Mix colors based on alpha ratio: $\tilde{\mathbf{c}}_i = \frac{\alpha_i \mathbf{c}_i + \alpha_i' \mathbf{c}_i'}{\alpha_i + \alpha_i'}$ [S1].
                *   Set $\tilde{\sigma}_i = \sigma_i$ (or mix densities similarly) [S1].
            *   If $\mathbf{x}_i'$ is not selected: Use standard NeRF output $(\sigma_i, \mathbf{c}_i)$ or apply specific transform logic as defined by the user [S1].
5.  **Volume Rendering:** Perform standard alpha-compositing volume rendering using the modified tuples $(\tilde{\sigma}_i, \tilde{\mathbf{c}}_i)$ [S1]:
    $$ C = \sum_{i=1}^N T_i \tilde{\alpha}_i \tilde{\mathbf{c}}_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \tilde{\alpha}_j) $$
    where $\tilde{\alpha}_i = 1 - e^{-\tilde{\sigma}_i \delta_i}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_editing_procedure(ray_origin, ray_direction, nerf_model, feature_field, 
                           transform_func=None, color_func=None, threshold=0.5):
    """
    Renders an edited view by intercepting NeRF sampling.
    
    Args:
        ray_origin: (3,) array, camera origin.
        ray_direction: (3,) array, camera direction.
        nerf_model: Function F_theta(x, v) -> (sigma, color).
        feature_field: Function that returns probability p(x) of belonging to target.
        transform_func: Optional function g_inv(x) for geometric edits.
        color_func: Optional function b(c) for color edits.
        threshold: Float, probability threshold for selection.
        
    Returns:
        edited_color: (3,) array, final pixel color.
    """
    # 1. Sample points along the ray
    t_vals = np.linspace(0.2, 1.0, 128) # Example sampling
    points = ray_origin + t_vals[:, None] * ray_direction
    
    # 2. Query feature field for all points
    probabilities = feature_field(points)
    
    # 3. Determine selection mask
    selected = probabilities >= threshold
    
    # 4. Process each point
    final_sigma = np.zeros_like(probabilities)
    final_color = np.zeros((len(points), 3))
    
    for i, (x, p, is_sel) in enumerate(zip(points, probabilities, selected)):
        if not is_sel:
            # Not selected: use vanilla NeRF
            sigma, c = nerf_model(x, ray_direction)
            final_sigma[i] = sigma
            final_color[i] = c
        else:
            # Selected: apply transforms
            sigma, c = nerf_model(x, ray_direction)
            
            if transform_func is not None:
                # Check for geometric transformation
                x_prime = transform_func(x)
                p_prime = feature_field(x_prime)
                
                if p_prime >= threshold:
                    # Both x and x_prime selected: mix colors
                    sigma_prime, c_prime = nerf_model(x_prime, ray_direction)
                    
                    # Compute alphas (assuming uniform delta for simplicity)
                    delta = t_vals[1] - t_vals[0]
                    alpha = 1.0 - np.exp(-sigma * delta)
                    alpha_prime = 1.0 - np.exp(-sigma_prime * delta)
                    
                    # Mix colors based on alpha ratio
                    total_alpha = alpha + alpha_prime
                    if total_alpha > 0:
                        mix_weight = alpha / total_alpha
                        c = mix_weight * c + (1.0 - mix_weight) * c_prime
                    # Density remains sigma (or could be mixed)
                    final_sigma[i] = sigma
                    final_color[i] = c
                else:
                    # x_prime not selected: use standard NeRF output for x
                    final_sigma[i] = sigma
                    final_color[i] = c
            else:
                # No geometric transform, apply color or deletion
                if color_func is not None:
                    c = color_func(c)
                # If deletion is intended, sigma is set to 0 (handled by caller or flag)
                final_sigma[i] = sigma
                final_color[i] = c
                
    # 5. Volume Rendering
    deltas = np.diff(t_vals, prepend=0.0) # Step sizes
    alphas = 1.0 - np.exp(-final_sigma * deltas)
    transmittance = np.cumprod(1.0 - alphas)
    transmittance = np.concatenate(([1.0], transmittance[:-1]))
    
    weights = alphas * transmittance
    edited_color = np.sum(weights[:, None] * final_color, axis=0)
    
    return edited_color
```

**Parameters:**
*   $T$ (Threshold): Typical value 0.5 [S1].
*   $\lambda$ (Warm-up blend ratio): Used in ViCA-NeRF warm-up, typical values not specified, but controls preservation of original pixel value [S2].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** When mixing colors at the boundary of a deformed object, discontinuities may occur if the alpha mixing is not smooth [S1].
*   **Feature Field Accuracy:** The quality of editing depends heavily on the accuracy of the external feature field (e.g., LSeg/DINO) or segmentation masks (SAM). Errors in $p(\mathbf{x})$ lead to incorrect selection [S1, S2].
*   **Occlusion Handling:** In geometric transformations, if the transformed point $\mathbf{x}'$ is occluded in the original scene, the NeRF query at $\mathbf{x}'$ may return background data, leading to artifacts [S1].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive initial masks, then projects them across views using depth to ensure multi-view consistency [S2]. This method is chosen when the user wants to isolate and modify a single object (e.g., "turn the shelf into metal") while keeping the background intact, addressing the pitfall where existing methods change both foreground and background [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit, projects the edited view onto all other views, and updates the training data using the formula $I_i' = \lambda I_i \odot m_w + (1 - \lambda)I_{i,j} \odot m_w + I_i \odot (1 - m_w)$, where $m_w$ is the binary mask and $\lambda$ controls the ratio [S2]. This is chosen to reduce the number of iterative updates required by methods like Instruct-NeRF2NeRF [S2].

### NeRF-RPN Application
A method that edits the scene by setting the density inside a proposal bounding box to zero when rendering [S3]. It uses a Region Proposal Network (RPN) adapted for NeRFs to detect objects and then performs deletion by masking out the density $\sigma$ within the 3D bounding box [S3]. This is chosen for object removal tasks where the object is detected via 3D proposals [S3].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ per pixel, where $N$ is the number of samples per ray and $M$ is the cost of querying the external feature field/segmentation model [S1, S2]. Since no backpropagation is performed, it is significantly faster than optimization-based methods like [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md) which require multiple iterations [S1].
*   **Memory Complexity:** $O(N)$ for storing sampled points and intermediate values per ray [S1]. The external feature field (e.g., LSeg) requires additional memory for the teacher network, but this is static [S1].
*   **Strengths:** Real-time or near-real-time editing; no retraining required; flexible integration of various priors (semantic, geometric) [S1, S2].
*   **Limitations:** Quality is limited by the accuracy of the external guidance (feature field/segmentation); may produce artifacts at boundaries or when transformations cause occlusions [S1, S2]. Not suitable for edits that require significant geometric changes not captured by the feature field [S1].
*   **When to use:** When real-time editing is needed, or when the user wants to apply semantic edits (color, deletion) without retraining [S1, S2].
*   **When not to use:** When high-fidelity geometric changes are required that are not well-represented by the feature field, or when the external guidance is unavailable or inaccurate [S1].

## Background
The NeRF Editing Procedure was introduced in "Decomposing NeRF for Editing Via Feature Field Distillation" (DFF) by [S1], which proposed using a feature field (distilled from LSeg/DINO) to guide editing at inference time. It builds on the foundational NeRF work [S1]. The method has been extended by ViCA-NeRF [S2] for view-consistency-aware editing and NeRF-RPN [S3] for object detection-based editing. It is used in applications requiring interactive 3D scene editing, such as virtual reality content creation and digital twin modification [S1, S2, S3].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
