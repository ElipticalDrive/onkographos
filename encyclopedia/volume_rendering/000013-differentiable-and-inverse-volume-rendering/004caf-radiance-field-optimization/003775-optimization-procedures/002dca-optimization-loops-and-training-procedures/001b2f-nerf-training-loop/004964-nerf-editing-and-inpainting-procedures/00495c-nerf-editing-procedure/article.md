# NeRF Editing Procedure

Also called: On-the-fly NeRF Editing, Feature-Guided NeRF Editing

## Overview
The NeRF Editing Procedure modifies the output of a trained Neural Radiance Field (NeRF) without retraining the underlying network parameters $\Theta$. It achieves this by intercepting the volume rendering pipeline to alter density $\sigma$ and color $\mathbf{c}$ values on-the-fly based on semantic or geometric queries [S1, S2, S3]. Unlike optimization-based methods that minimize a loss function over many iterations [12372-nerf-in-inpainting-optimization.md], this approach applies immediate transformations during ray marching, enabling real-time or near-real-time editing such as deletion, colorization, and geometric transformation [S1].

## How it works
The method operates by modifying the standard NeRF rendering loop. Instead of querying the network $f_\theta(\mathbf{x}, \mathbf{d})$ directly for all points, it introduces a selection mechanism driven by an external feature field or mask.

### Inputs and Outputs
*   **Inputs:**
    *   Ray $r$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
    *   Trained NeRF network $f_\theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
    *   Editing Query/Model: A mechanism to determine if a 3D coordinate $\mathbf{x}$ belongs to the target edit region. This is typically a Distilled Feature Field (DFF) [S1], a segmentation mask [S2], or a bounding box proposal [S3].
    *   Editing Parameters: Thresholds for selection, transformation functions (e.g., colorization $b$), and geometric inverse maps $g^{-1}$.
*   **Outputs:**
    *   Edited pixel color $\hat{C}(r)$.

### Data Structures
1.  **NeRF Network $f_\theta$:** Standard MLP taking position $\mathbf{x}$ and direction $\mathbf{d}$, outputting scalar density $\sigma \in \mathbb{R}$ and RGB color $\mathbf{c} \in \mathbb{R}^3$.
2.  **Feature Field / Query Model:**
    *   *DFF (from [S1]):* A network or lookup structure that outputs a probability $p \in [0, 1]$ indicating the likelihood that coordinate $\mathbf{x}$ matches a semantic query (e.g., "apple").
    *   *Mask/Proposal (from [S2, S3]):* Binary masks or bounding boxes defining the edit region.
3.  **Ray Sample Points:** A list of 3D coordinates $\{\mathbf{x}_1, \dots, \mathbf{x}_N\}$ sampled along the ray.

### Numbered Steps
1.  **Sample Points:** Sample $N$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_N\}$ along the ray $r$ at depths $t_i$, as in standard NeRF rendering [S1].
2.  **Query Selection:** For each point $\mathbf{x}_i$:
    *   Query the editing model to determine if $\mathbf{x}_i$ is "selected" by the user's intent.
    *   *If using DFF [S1]:* Calculate probability $p_i$. If $p_i > \tau$ (user-defined threshold), mark $\mathbf{x}_i$ as selected. Alternatively, use $p_i$ as a soft weight [S1].
    *   *If using Mask/Box [S2, S3]:* Check if $\mathbf{x}_i$ lies within the projected 2D mask or 3D bounding box.
3.  **Compute Raw Values:** Query the NeRF $f_\theta$ to obtain raw density $\sigma_i = \sigma(\mathbf{x}_i)$ and raw color $\mathbf{c}_i = \mathbf{c}(\mathbf{x}_i, \mathbf{d})$ [S1].
4.  **Apply Transformations:** Modify $\sigma_i$ and $\mathbf{c}_i$ based on the selection status:
    *   **Case A: Not Selected:** Keep $\sigma_i$ and $\mathbf{c}_i$ unchanged [S1].
    *   **Case B: Selected:** Apply one of the following transforms [S1]:
        *   **Deletion:** Set $\sigma_i \leftarrow 0$. This makes the point transparent, effectively removing the object [S1, S3].
        *   **Color Editing:** Apply a colorization function $b(\cdot)$ to the color: $\mathbf{c}_i \leftarrow b(\mathbf{c}_i)$. The density $\sigma_i$ remains unchanged [S1].
        *   **Translation/Rescaling:**
            1.  Compute the inverse geometric transformation $\mathbf{x}_0 = g^{-1}(\mathbf{x}_i)$ [S1].
            2.  Check if $\mathbf{x}_0$ is also selected by the query.
            3.  If $\mathbf{x}_0$ is selected, query the NeRF for $\sigma_0, \mathbf{c}_0$ at $\mathbf{x}_0$.
            4.  If both $\mathbf{x}_i$ and $\mathbf{x}_0$ are selected and have non-zero density, mix the colors based on their alphas:
                $$ \mathbf{c}_{final} = \frac{\alpha_i \mathbf{c}_i + \alpha_0 \mathbf{c}_0}{\alpha_i + \alpha_0} $$
                where $\alpha$ is derived from $\sigma$ and step size $\delta$ [S1].
            5.  If only $\mathbf{x}_i$ is selected (or $\mathbf{x}_0$ is not selected), use $\mathbf{c}_i$ (or $\mathbf{c}_0$ if $\mathbf{x}_0$ is selected but $\mathbf{x}_i$ is not, depending on implementation logic for displacement) [S1].
5.  **Volume Rendering:** Render the ray using the modified tuples $(\sigma_i, \mathbf{c}_i)$:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i \mathbf{c}_i $$
    where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1, 12491-nerf-editing-procedure.md].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray_origin, ray_direction, nerf_model, edit_query, edit_params):
    """
    Renders a single ray with on-the-fly editing.
    
    Args:
        ray_origin: 3D vector
        ray_direction: 3D vector
        nerf_model: Function f_theta(x, d) -> (sigma, color)
        edit_query: Function query(x) -> (is_selected, probability)
        edit_params: Dict containing threshold, colorization_func, inverse_transform
        
    Returns:
        edited_color: 3D vector
    """
    # 1. Sample points along the ray
    t_min, t_max = edit_params['t_min'], edit_params['t_max']
    num_samples = edit_params['num_samples']
    t_vals = np.linspace(t_min, t_max, num_samples)
    points = ray_origin + t_vals[:, None] * ray_direction
    
    # Pre-compute alphas for mixing if needed
    alphas = []
    colors = []
    sigmas = []
    
    for i, x in enumerate(points):
        # 2. Query selection
        is_selected, prob = edit_query(x)
        
        # 3. Get raw NeRF values
        sigma_raw, color_raw = nerf_model(x, ray_direction)
        
        # 4. Apply Transformations
        if not is_selected:
            # Case A: No edit
            sigma_final = sigma_raw
            color_final = color_raw
        else:
            # Case B: Selected
            transform_type = edit_params['transform_type']
            
            if transform_type == 'deletion':
                # Deletion: Set density to zero
                sigma_final = 0.0
                color_final = color_raw # Color irrelevant if sigma is 0
                
            elif transform_type == 'color_edit':
                # Color Editing: Transform color
                color_func = edit_params['colorization_func']
                sigma_final = sigma_raw
                color_final = color_func(color_raw)
                
            elif transform_type == 'translation':
                # Translation/Rescaling
                inverse_transform = edit_params['inverse_transform']
                x_deformed = inverse_transform(x)
                
                # Check if deformed point is also selected
                is_deformed_selected, _ = edit_query(x_deformed)
                
                if is_deformed_selected:
                    # Query NeRF for deformed point
                    sigma_deformed, color_deformed = nerf_model(x_deformed, ray_direction)
                    
                    # Calculate alpha for mixing (simplified from standard volume rendering)
                    # Note: delta is constant step size in this simple loop
                    delta = t_vals[1] - t_vals[0] if len(t_vals) > 1 else 0.1
                    alpha_i = 1.0 - np.exp(-sigma_raw * delta)
                    alpha_0 = 1.0 - np.exp(-sigma_deformed * delta)
                    
                    # Mix colors based on alpha ratio
                    total_alpha = alpha_i + alpha_0
                    if total_alpha > 0:
                        color_final = (alpha_i * color_raw + alpha_0 * color_deformed) / total_alpha
                    else:
                        color_final = color_raw
                        
                    # Density handling for mixing: 
                    # The source [S1] specifies mixing colors in ratio of alphas.
                    # Density is typically kept as the sum or max for transparency, 
                    # but for simplicity in on-the-fly, we might keep sigma_raw 
                    # or sum them. Source implies color mixing is the key.
                    sigma_final = sigma_raw + sigma_deformed 
                else:
                    # Only original point selected
                    sigma_final = sigma_raw
                    color_final = color_raw
            else:
                sigma_final = sigma_raw
                color_final = color_raw
                
        sigmas.append(sigma_final)
        colors.append(color_final)
        
    # 5. Volume Rendering
    sigmas = np.array(sigmas)
    colors = np.array(colors)
    delta = t_vals[1] - t_vals[0] if len(t_vals) > 1 else 0.1
    
    # Compute transmittance T_i
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta)
    # We can compute cumulative sum of sigma * delta
    sigma_delta = sigmas * delta
    cum_sigma_delta = np.cumsum(sigma_delta)
    
    # T_0 = 1.0, T_i = exp(-cum_sigma_delta[i-1])
    # For i=0, T_0 = 1.0
    # For i>0, T_i = exp(-cum_sigma_delta[i-1])
    
    transmittance = np.ones(num_samples)
    if num_samples > 1:
        transmittance[1:] = np.exp(-cum_sigma_delta[:-1])
        
    # Weights w_i = T_i * (1 - exp(-sigma_i * delta))
    alpha_i = 1.0 - np.exp(-sigma_delta)
    weights = transmittance * alpha_i
    
    # Final color
    edited_color = np.sum(weights[:, None] * colors, axis=0)
    
    return edited_color
```

### Parameters and Typical Values
*   **Threshold $\tau$:** User-defined probability threshold for DFF selection. Not given in the sources.
*   **$\lambda$ (Warm-up):** Hyperparameter controlling the ratio preserved from the original pixel value in ViCA-NeRF warm-up. Typical values not specified, but used in blending formula [S2].
*   **Colorization Function $b$:** Depends on the editing task (e.g., Instruct-Pix2Pix for style transfer) [S2].

### Edge Cases and Pitfalls
*   **Boundary Mixing:** When translating objects, if both the original and deformed points are selected, simple alpha mixing is used. This may cause artifacts at object boundaries if the geometry is complex [S1].
*   **Depth Propagation Errors:** In local editing using SAM and depth, incorrect depth estimation can lead to masks projecting incorrectly across views, causing inconsistent edits [S2].
*   **Background Alteration:** Existing methods may inadvertently change the background if the segmentation mask is not precise. ViCA-NeRF addresses this by integrating wrapped segmentation masks [S2].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive masks, allows users to pinpoint targets with a single point, and projects these masks across views using depth data. It continues projection as long as the projected mask sufficiently overlaps with the current view's mask, integrating the wrapped mask with current segmented masks. This ensures exclusive modification of the target object (e.g., a human figure) while leaving the background intact, addressing the pitfall of simultaneous background/foreground changes in other methods [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views. The update for a view $i$ given an edited view $j$ is calculated as $I_i' = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w)$, where $m_w$ is the binary mask for the projected area and $\lambda$ controls the ratio preserved from the original pixel value [S2].

### NeRF-RPN Application
A method to edit the scene by setting the density inside a proposal bounding box to zero when rendering [S3]. It utilizes NeRF-RPN, a framework for object detection in NeRFs, to generate proposals for first-level objects. The editing is performed by masking out the density of points falling within these bounding boxes, effectively removing the detected objects from the scene [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. The additional cost is the query to the editing model (DFF, SAM, or RPN) for each sample. DFF queries are typically fast if distilled, but SAM-based propagation requires depth estimation and projection [S1, S2].
*   **Memory Complexity:** Depends on the editing model. DFF requires storing feature maps or a distilled network [S1]. SAM requires storing segmentation masks for all views [S2].
*   **Strengths:** Immediate feedback without iterative optimization; ability to edit specific semantic parts; supports various edit types (deletion, color, geometry) [S1, S2].
*   **Limitations:** Quality depends on the accuracy of the editing model (DFF, SAM, RPN); geometric edits may cause artifacts at boundaries; depth propagation errors can affect local editing consistency [S1, S2].
*   **When to Use:** Real-time editing, semantic editing, or when iterative optimization is too slow [S1, S2].
*   **When Not to Use:** When high-fidelity geometric consistency is required without artifacts, or when the editing model fails to segment the target accurately [S2].

## Background
The NeRF Editing Procedure builds on the foundational NeRF optimization [12491-nerf-editing-procedure.md] by introducing on-the-fly modifications. The concept of using feature fields for editing was introduced in "Decomposing NeRF for Editing Via Feature Field Distillation" (DFF) [S1], which distills semantic information into a feature field to guide editing. ViCA-NeRF [S2] extended this with view-consistency-aware editing and warm-up strategies. NeRF-RPN [S3] applied object detection proposals for editing. These methods represent a shift from purely optimization-based editing to direct manipulation of the radiance field representation.

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
