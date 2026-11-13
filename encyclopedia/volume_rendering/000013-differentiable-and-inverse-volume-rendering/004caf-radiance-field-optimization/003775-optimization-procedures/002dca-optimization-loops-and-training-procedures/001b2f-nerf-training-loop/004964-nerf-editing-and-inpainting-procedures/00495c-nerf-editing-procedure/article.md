# NeRF Editing Procedure

Also called: Coordinate-based Editing, Feature Field Editing, DFF-based Editing

## Overview
The NeRF Editing Procedure is a method for modifying a trained Neural Radiance Field ($f_\theta$) by integrating external semantic or geometric guidance directly into the volume rendering pipeline. Unlike methods that retrain the network from scratch or rely solely on 2D image inpainting, this approach uses a Distilled Feature Field (DFF) or similar query mechanism to determine *where* edits should occur in 3D space. It solves the problem of localized 3D editing (such as deletion, colorization, or translation) while maintaining multi-view consistency by applying transformations to the density and color values of sampled points based on their semantic probability. It differs from [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) by operating on the continuous radiance field parameters rather than discrete pixel buffers, and from [Semantic and Language-Guided Editing](14393-semantic-and-language-guided-editing.md) by explicitly defining the editing logic through coordinate selection probabilities rather than just loss constraints.

## How it works
The method modifies the standard volume rendering equation by intercepting the density $\sigma$ and color $\mathbf{c}$ of sampled points and applying conditional transforms based on a query probability $p$.

**Inputs:**
*   Trained NeRF model $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
*   Distilled Feature Field (DFF) or query encoder $Q(\mathbf{x})$ that outputs a probability $p \in [0, 1]$ indicating the likelihood that coordinate $\mathbf{x}$ matches a user-defined target (e.g., "apple", "skull", or a specific object mask).
*   Editing transformation parameters (e.g., translation vector, colorization function).
*   User-defined threshold $\tau$ for coordinate selection.

**Outputs:**
*   Edited rendered images $\hat{C}_{edited}(\mathbf{r})$ for input rays $\mathbf{r}$.

**Data Structures:**
*   **Ray Sample Points:** A list of 3D coordinates $\{\mathbf{x}_1, \dots, \mathbf{x}_N\}$ sampled along a ray $\mathbf{r}$, typically using stratified sampling between near and far bounds.
*   **Probability Map:** For each point $\mathbf{x}_i$, a scalar probability $p_i$ returned by the DFF.

**Numbered Steps:**

1.  **Ray Sampling:** For a given camera ray $\mathbf{r}$, sample $N$ points $\mathbf{x}_i$ along the ray as in standard NeRF rendering [S1].
2.  **Query Probability Calculation:** For each point $\mathbf{x}_i$, query the DFF to obtain the probability $p_i$ that $\mathbf{x}_i$ belongs to the target semantic region [S1].
3.  **Coordinate Selection:** Define a binary selection state $s_i$ for each point:
    $$
    s_i = \begin{cases} 
    1 & \text{if } p_i \geq \tau \\
    0 & \text{if } p_i < \tau 
    \end{cases}
    $$
    Alternatively, a soft mixture can be used where the contribution is weighted by $p_i$ [S1].
4.  **Transform Application:** For each point $\mathbf{x}_i$, determine the final density $\sigma'_i$ and color $\mathbf{c}'_i$ based on $s_i$:
    *   **If $s_i = 0$ (Not Selected):** The point is unaffected. Query the vanilla NeRF:
        $$
        \sigma'_i = \sigma(\mathbf{x}_i), \quad \mathbf{c}'_i = \mathbf{c}(\mathbf{x}_i)
        $$
    *   **If $s_i = 1$ (Selected):** Apply the specific editing transform [S1]:
        *   **Deletion:** Set density to zero to remove the object:
            $$
            \sigma'_i = 0, \quad \mathbf{c}'_i = \mathbf{c}(\mathbf{x}_i) \quad (\text{color is irrelevant as } \sigma=0)
            $$
        *   **Color Editing:** Query the NeRF for base values, then apply a colorization function $b(\cdot)$:
            $$
            \sigma'_i = \sigma(\mathbf{x}_i), \quad \mathbf{c}'_i = b(\mathbf{c}(\mathbf{x}_i))
            $$
        *   **Translation/Rescaling:** Compute the inverse transformation $g^{-1}$ of the editing operation. Let $\mathbf{x}_0 = g^{-1}(\mathbf{x}_i)$.
            *   Query the DFF for $\mathbf{x}_0$ to get $p_0$.
            *   If $p_0 \geq \tau$ (the deformed location is also part of the target), use $\mathbf{x}_0$ to query the NeRF for density $\sigma_0$ and color $\mathbf{c}_0$.
            *   If both $\mathbf{x}_i$ and $\mathbf{x}_0$ are selected and have non-zero density (e.g., at boundaries), mix the colors based on their alphas $\alpha_i$ and $\alpha_0$:
                $$
                \mathbf{c}'_i = \frac{\alpha_i \mathbf{c}(\mathbf{x}_i) + \alpha_0 \mathbf{c}_0}{\alpha_i + \alpha_0}
                $$
            *   Otherwise, use the values from the selected coordinate [S1].
5.  **Volume Rendering:** Perform standard volume rendering using the modified tuples $(\sigma'_i, \mathbf{c}'_i)$ to compute the final pixel color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray_origin, ray_direction, nerf_model, dff_model, 
                      transform_func, colorization_func, threshold, 
                      edit_type='deletion'):
    """
    Renders a single ray with editing applied via DFF queries.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        nerf_model: Function f_theta(x, d) -> (sigma, c).
        dff_model: Function Q(x) -> p (probability of matching query).
        transform_func: Function g_inv(x) -> x0 (inverse geometric transform).
        colorization_func: Function b(c) -> c_edited.
        threshold: Float, probability threshold tau for selection.
        edit_type: String, 'deletion', 'color', or 'translation'.
        
    Returns:
        color: 3D vector, final pixel color.
    """
    # 1. Sample points along the ray (standard NeRF sampling)
    # Assuming a helper function that returns N points and their depths
    points, depths = sample_points_along_ray(ray_origin, ray_direction, num_samples=128)
    
    # Lists to store modified sigma and color
    sigmas = []
    colors = []
    
    for i, x in enumerate(points):
        # 2. Query DFF for probability
        p = dff_model(x)
        
        # 3. Coordinate Selection
        is_selected = (p >= threshold)
        
        if not is_selected:
            # 4a. Not Selected: Use vanilla NeRF
            sigma, c = nerf_model(x, ray_direction)
            sigmas.append(sigma)
            colors.append(c)
        else:
            # 4b. Selected: Apply Transform
            if edit_type == 'deletion':
                # 4b-i. Deletion: Set density to zero
                sigmas.append(0.0)
                colors.append([0.0, 0.0, 0.0]) # Color irrelevant
                
            elif edit_type == 'color':
                # 4b-ii. Color Editing: Transform color
                sigma, c = nerf_model(x, ray_direction)
                c_edited = colorization_func(c)
                sigmas.append(sigma)
                colors.append(c_edited)
                
            elif edit_type == 'translation':
                # 4b-iii. Translation/Rescaling
                # Compute deformed coordinate
                x0 = transform_func(x)
                
                # Query DFF for deformed coordinate
                p0 = dff_model(x0)
                is_selected_x0 = (p0 >= threshold)
                
                # Query NeRF for original point
                sigma_x, c_x = nerf_model(x, ray_direction)
                
                if is_selected_x0:
                    # Query NeRF for deformed point
                    sigma_x0, c_x0 = nerf_model(x0, ray_direction)
                    
                    # Calculate alphas for mixing (assuming standard alpha formula)
                    # alpha = 1 - exp(-sigma * delta)
                    # Note: delta is depth difference between samples, approximated here
                    # For simplicity in this self-contained pseudocode, we assume 
                    # alpha is proportional to sigma for the mixing ratio as per source
                    alpha_x = 1.0 - math.exp(-sigma_x) 
                    alpha_x0 = 1.0 - math.exp(-sigma_x0)
                    
                    # Mix colors in ratio of alphas
                    total_alpha = alpha_x + alpha_x0
                    if total_alpha > 0:
                        c_mixed = (alpha_x * c_x + alpha_x0 * c_x0) / total_alpha
                    else:
                        c_mixed = c_x
                        
                    sigmas.append(sigma_x) # Keep original sigma or mix? Source implies mixing colors
                    colors.append(c_mixed)
                else:
                    # x0 not selected, use x values but potentially transformed?
                    # Source says "If x0 is selected... x0 is used instead".
                    # If not selected, we fall back to standard behavior for x?
                    # Or does the transform apply regardless? 
                    # Source: "If x0 is selected by the query, x0 is used instead of x"
                    # Implies if NOT selected, we use x.
                    sigmas.append(sigma_x)
                    colors.append(c_x)
            else:
                # Default: Vanilla
                sigma, c = nerf_model(x, ray_direction)
                sigmas.append(sigma)
                colors.append(c)

    # 5. Volume Rendering
    # Standard alpha compositing
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    
    for sigma, c in zip(sigmas, colors):
        alpha = 1.0 - math.exp(-sigma) # Simplified alpha calculation
        weight = alpha * (1.0 - accumulated_alpha)
        accumulated_color += weight * c
        accumulated_alpha += weight
        
    return accumulated_color
```

**Parameters:**
*   $\tau$ (Threshold): User-defined probability threshold for coordinate selection. Typical values are not explicitly given in the sources, but the source notes a label set for "skipping tuning of thresholds" [S1], implying a default or automated selection might be used.
*   $\lambda$ (Warm-up ratio): Used in ViCA-NeRF warm-up strategy, controls the ratio preserved from the original pixel value [S2].

**Edge Cases and Pitfalls:**
*   **Boundary Mixing:** When translating objects, points near the boundary may have both $\mathbf{x}$ and $\mathbf{x}_0$ selected. The source specifies mixing colors in the ratio of their alphas to handle this [S1].
*   **Threshold Sensitivity:** The choice of $\tau$ affects the precision of the edit. A high threshold may miss parts of the object; a low threshold may include background noise [S1].
*   **Depth Propagation Errors:** In local editing variants, incorrect depth estimation can lead to masks projecting incorrectly across views, causing artifacts [S2].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information. It employs the Segment Anything Model (SAM) to derive masks, allows users to pinpoint a target instance with a single point, and projects the mask across views using depth information until sufficient overlap is achieved. This ensures multi-view consistency and prevents simultaneous changes to the background and foreground [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency. It randomly selects a view to edit and projects the edited view onto all other views. The update for a view $i$ given an edited view $j$ is calculated as $I'_i = \lambda I_i \odot m_w + (1 - \lambda)I_{i,j} \odot m_w + I_i \odot (1 - m_w)$, where $m_w$ is the binary mask for the projected area and $\lambda$ controls the ratio preserved from the original pixel value [S2].

### NeRF-RPN Application
An application of the editing procedure using Region Proposal Networks (RPN) for object detection in NeRFs. It edits the scene by setting the density inside a proposal bounding box to zero when rendering, effectively removing detected objects [S3].

## Operating characteristics
*   **Time Complexity:** The method adds a query step to the DFF for every sampled point during rendering. This increases the per-ray rendering time by a factor proportional to the cost of the DFF query relative to the NeRF query. Training time is not directly affected as this is an inference-time editing procedure, though the DFF itself requires training [S1].
*   **Memory Complexity:** Requires storing the DFF parameters in addition to the NeRF parameters. The DFF is typically a lightweight feature field, so the memory overhead is moderate [S1].
*   **Strengths:** Allows for precise, semantic-based editing (deletion, colorization, transformation) without retraining the entire NeRF. Maintains multi-view consistency by operating in 3D space [S1].
*   **Limitations:** Relies on the accuracy of the DFF and the quality of the segmentation/query probabilities. Geometric transformations can introduce artifacts at object boundaries if not handled with alpha mixing [S1].
*   **When to use:** When precise, localized edits are required without retraining, or when semantic guidance (e.g., "remove the chair") is available.
*   **When not to use:** When the DFF is not available or inaccurate, or when global structural changes are needed that cannot be captured by local coordinate transforms.

## Background
The NeRF Editing Procedure via Feature Field Distillation was introduced by [S1] (Decomposing NeRF for Editing Via Feature Field Distillation). It builds on the concept of distilling semantic features from pre-trained models (like LSeg or DINO) into a field that can be queried during rendering. This allows for editing operations that are consistent with the semantic content of the scene. The method was developed to address the limitations of previous editing techniques that either required iterative retraining or lacked 3D consistency.

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
