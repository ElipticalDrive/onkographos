# NeRF Editing Procedure

Also called: DFF Editing, Coordinate-Selected Editing

## Overview
The NeRF Editing Procedure modifies a radiance field $F_\Theta(\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$ by selectively transforming density and color values during volume rendering based on semantic or geometric selection criteria [S1]. It distinguishes itself from global optimization methods by operating as a differentiable rendering-time transform, allowing edits (deletion, colorization, translation) without retraining the underlying NeRF weights [S1]. It builds on [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md) by utilizing a Distilled Feature Field (DFF) to define regions of interest (ROI) via coordinate probability [S1].

## How it works
**Inputs:**
1.  A trained vanilla NeRF $F_\Theta$ parameterized by $\Theta$, outputting density $\sigma$ and color $\mathbf{c}$ for any 3D coordinate $\mathbf{x}$ and view direction $\mathbf{v}$ [S1].
2.  A Distilled Feature Field (DFF) or equivalent semantic probability field $P(\mathbf{x})$ that outputs a probability $p \in [0, 1]$ indicating the likelihood that coordinate $\mathbf{x}$ belongs to a target semantic class or object [S1].
3.  An editing transformation function $g(\mathbf{x})$ (e.g., translation vector, scaling factor) and its inverse $g^{-1}(\mathbf{x})$ [S1].
4.  A colorization function $b(\mathbf{c})$ that maps input color to target color [S1].
5.  A user-defined probability threshold $\tau$ for coordinate selection [S1].

**Outputs:**
*   Rendered image pixels $\mathbf{I}$ computed via volume rendering using the modified $(\sigma, \mathbf{c})$ tuples [S1].

**Data Structures:**
*   **NeRF Model:** Standard MLP or grid-based structure mapping $(\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$.
*   **DFF/Probability Field:** A function or grid mapping $\mathbf{x} \to p$. In the DFF implementation, this is derived from teacher networks (LSeg or DINO) distilled into a coordinate-based field [S1].
*   **Ray:** A sequence of sampled points $\{\mathbf{x}_1, \mathbf{x}_2, \dots, \mathbf{x}_N\}$ along a camera ray [S1].

**Steps:**
1.  **Ray Sampling:** For each pixel, sample $N$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_N\}$ along the viewing ray, including near and far clipping planes [S1].
2.  **Coordinate Selection:** For each point $\mathbf{x}_i$:
    *   Query the DFF to obtain probability $p_i = P(\mathbf{x}_i)$ [S1].
    *   Determine if the coordinate is "selected" by the query:
        *   If $p_i \ge \tau$, the coordinate is **selected** [S1].
        *   If $p_i < \tau$, the coordinate is **not selected** [S1].
        *   *Note:* The source also mentions using a mixture of selected and not-selected results proportional to $p$ without a hard threshold, but the primary procedural definition uses the threshold $\tau$ [S1].
3.  **Value Transformation:** For each point $\mathbf{x}_i$, compute the effective density $\sigma'_i$ and color $\mathbf{c}'_i$:
    *   **Case A: Not Selected:**
        *   Query the vanilla NeRF: $(\sigma_i, \mathbf{c}_i) = F_\Theta(\mathbf{x}_i, \mathbf{v})$ [S1].
        *   Set $\sigma'_i = \sigma_i$ and $\mathbf{c}'_i = \mathbf{c}_i$ [S1].
    *   **Case B: Selected:** Apply the specific editing transform:
        *   **Deletion:** Set $\sigma'_i = 0$. The color $\mathbf{c}'_i$ is irrelevant as density is zero [S1].
        *   **Color Editing:** Query the vanilla NeRF for $(\sigma_i, \mathbf{c}_i) = F_\Theta(\mathbf{x}_i, \mathbf{v})$. Transform the color: $\mathbf{c}'_i = b(\mathbf{c}_i)$. Keep $\sigma'_i = \sigma_i$ [S1].
        *   **Translation / Rescaling:**
            1.  Compute the deformed coordinate $\mathbf{x}_0 = g^{-1}(\mathbf{x}_i)$ [S1].
            2.  Check if $\mathbf{x}_0$ is also selected by the query (i.e., $P(\mathbf{x}_0) \ge \tau$) [S1].
            3.  If $\mathbf{x}_0$ is selected:
                *   Query the NeRF for both points: $(\sigma_i, \mathbf{c}_i) = F_\Theta(\mathbf{x}_i, \mathbf{v})$ and $(\sigma_0, \mathbf{c}_0) = F_\Theta(\mathbf{x}_0, \mathbf{v})$ [S1].
                *   If both $\sigma_i > 0$ and $\sigma_0 > 0$ (e.g., at boundaries where the object overlaps with itself or others), mix the colors based on their alphas:
                    $$ \alpha_i = 1 - e^{-\sigma_i \Delta t}, \quad \alpha_0 = 1 - e^{-\sigma_0 \Delta t} $$
                    $$ \mathbf{c}'_i = \frac{\alpha_i \mathbf{c}_i + \alpha_0 \mathbf{c}_0}{\alpha_i + \alpha_0} $$
                    $$ \sigma'_i = \sigma_i + \sigma_0 \quad \text{(or simply use } \sigma_i \text{ if only color mixing is intended for the visual blend, but the source implies mixing colors in ratio of alphas)} $$
                    *Clarification from source:* The source states "mix their colors $c(x)$ and $c(x_0)$ in the ratio of their alphas at the point for simplicity" [S1]. It does not explicitly redefine $\sigma$ in this specific boundary case, but implies the visual result is a blend. Standard volume rendering uses $\sigma$ for opacity. If $\sigma$ is not blended, the opacity might be incorrect. However, the text says "mix their colors... for simplicity". We assume $\sigma'_i$ remains $\sigma_i$ or is handled by the standard accumulation, but the color is the blended value [S1].
            4.  If $\mathbf{x}_0$ is **not** selected:
                *   Use $\mathbf{x}_i$ for standard NeRF query: $(\sigma'_i, \mathbf{c}'_i) = F_\Theta(\mathbf{x}_i, \mathbf{v})$ [S1].
                *   *Correction:* The source says "If $x_0$ is selected by the query, $x_0$ is used instead of $x$ for calculating color and density." This implies if $x_0$ is selected, we use the properties of $x_0$. If $x_0$ is NOT selected, we fall back to $x$? The source logic for Translation is: "If $x_0$ is selected... $x_0$ is used... If both $x$ and $x_0$ are selected... mix". It does not explicitly state the fallback if $x_0$ is not selected but $x$ is. However, standard translation implies moving the object. If $x$ is selected (part of object) and $x_0$ (the source location) is not selected (background), we should use the properties of $x_0$ (background) at location $x$? No, the NeRF is queried at $x_0$. So $(\sigma'_i, \mathbf{c}'_i) = F_\Theta(\mathbf{x}_0, \mathbf{v})$ [S1].
4.  **Volume Rendering:** Perform standard volume rendering using the series of transformed tuples $(\sigma'_i, \mathbf{c}'_i)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_edited_ray(ray_origin, ray_direction, nerf_model, dff_field, 
                      edit_type, transform_g_inv, color_func_b, threshold_tau):
    """
    Renders a single ray with NeRF editing applied.
    
    Args:
        ray_origin: (3,) array, camera origin.
        ray_direction: (3,) array, normalized ray direction.
        nerf_model: Function (x, v) -> (sigma, c).
        dff_field: Function (x) -> p (probability).
        edit_type: String, one of 'deletion', 'color', 'translation'.
        transform_g_inv: Function (x) -> x0 (inverse transform).
        color_func_b: Function (c) -> c' (colorization).
        threshold_tau: Float, probability threshold for selection.
        
    Returns:
        color: (3,) array, final pixel color.
    """
    # 1. Sample points
    t_vals = np.linspace(0.05, 1.0, 128) # Example sampling
    points = ray_origin + ray_direction * t_vals[:, None]
    
    sigma_accum = 0.0
    color_accum = np.zeros(3)
    alpha_accum = 0.0
    
    for i, x in enumerate(points):
        # 2. Coordinate Selection
        p = dff_field(x)
        is_selected = p >= threshold_tau
        
        sigma_val = 0.0
        c_val = np.zeros(3)
        
        if not is_selected:
            # Case A: Not Selected
            sigma_val, c_val = nerf_model(x, ray_direction)
        else:
            # Case B: Selected
            if edit_type == 'deletion':
                # 4-A: Deletion
                sigma_val = 0.0
                c_val = np.zeros(3) # Irrelevant
            elif edit_type == 'color':
                # 4-B: Color editing
                sigma_val, c_val = nerf_model(x, ray_direction)
                c_val = color_func_b(c_val)
            elif edit_type == 'translation':
                # 4'-C: Translation / Rescaling
                x0 = transform_g_inv(x)
                p0 = dff_field(x0)
                is_x0_selected = p0 >= threshold_tau
                
                if is_x0_selected:
                    # Query both
                    sigma_x, c_x = nerf_model(x, ray_direction)
                    sigma_x0, c_x0 = nerf_model(x0, ray_direction)
                    
                    if sigma_x > 0 and sigma_x0 > 0:
                        # Mix colors in ratio of alphas
                        # Note: delta_t is assumed constant or derived from t_vals
                        # For simplicity, assume equal spacing or use actual delta_t
                        delta_t = t_vals[i] - t_vals[i-1] if i > 0 else 0.05
                        alpha_x = 1.0 - np.exp(-sigma_x * delta_t)
                        alpha_x0 = 1.0 - np.exp(-sigma_x0 * delta_t)
                        
                        total_alpha = alpha_x + alpha_x0
                        if total_alpha > 0:
                            c_val = (alpha_x * c_x + alpha_x0 * c_x0) / total_alpha
                        else:
                            c_val = c_x
                        sigma_val = sigma_x # Source doesn't specify sigma blend, keeping x's sigma
                    else:
                        # Fallback if one is zero density
                        sigma_val, c_val = nerf_model(x0, ray_direction)
                else:
                    # x0 not selected, use x0 properties at x location
                    sigma_val, c_val = nerf_model(x0, ray_direction)
        
        # 3. Volume Rendering Accumulation
        alpha = 1.0 - np.exp(-sigma_val * 0.05) # Approximate delta_t
        weight = alpha * (1.0 - alpha_accum)
        color_accum += weight * c_val
        alpha_accum += weight
        
        if alpha_accum > 0.99:
            break
            
    return color_accum
```

**Parameters:**
*   $\tau$ (threshold): User-defined probability threshold. Typical values not given in sources, but must be $[0, 1]$ [S1].
*   $\lambda$ (warm-up ratio): Used in ViCA-NeRF warm-up, not the core DFF procedure [S2].
*   $b(\cdot)$: Colorization function. Implementation depends on the specific edit (e.g., CLIP-based colorization, simple hue shift) [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Mixing:** When translating objects, boundaries where the original and target positions overlap require alpha-weighted color mixing to avoid artifacts [S1].
*   **Threshold Sensitivity:** The choice of $\tau$ determines the edit region. A high threshold may miss parts of the object; a low threshold may include background noise [S1].
*   **Inverse Transform Validity:** For translation/rescaling, $g^{-1}(\mathbf{x})$ must be well-defined for all sampled points [S1].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive initial masks, where users pinpoint a target instance with a single point [S2]. The mask is then projected across different views using depth information, continuing as long as the projected mask sufficiently overlaps with the corresponding mask in the current view [S2]. This integrated mask is used to isolate the region for editing, preventing simultaneous changes to background and foreground [S2]. This method is chosen when precise object isolation is required without semantic probability fields [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire iterative editing process, accelerating convergence while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views [S2]. The update for a view $i$ given an edited view $j$ is calculated as:
$$ I'_i = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w) $$
where $m_w$ is the binary mask for the projected area and $\lambda$ is a hyperparameter controlling the ratio preserved from the original pixel value [S2]. This is used to provide a strong initial prior for subsequent refinement [S2].

### NeRF-RPN Application
An application of the editing procedure using 3D object detection proposals [S3]. The scene is edited by setting the density inside a proposal bounding box to zero when rendering [S3]. This effectively removes detected objects from the scene [S3]. It is used for object removal tasks where 3D bounding boxes are available from a detector like NeRF-RPN [S3].

## Operating characteristics
*   **Time Complexity:** The editing procedure adds negligible overhead to the rendering time, as it primarily involves querying the DFF and potentially an additional NeRF query for translation [S1]. The DFF query is typically a lightweight MLP evaluation [S1].
*   **Memory Complexity:** Requires storing the DFF weights in addition to the NeRF weights [S1]. The DFF is distilled from teacher networks (LSeg/DINO), so it is smaller than the full teacher models [S1].
*   **Performance:** Allows for real-time or near-real-time editing if the DFF is efficient [S1]. The quality depends on the accuracy of the DFF in representing semantic boundaries [S1].
*   **Strengths:** No retraining required; supports multiple edit types (deletion, color, translation) simultaneously; differentiable.
*   **Limitations:** Relies on the quality of the DFF; may produce artifacts at object boundaries if not handled correctly (e.g., alpha mixing) [S1].

## Background
The NeRF Editing Procedure was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" (DFF) [S1]. It builds on the concept of decomposing the NeRF into a geometry/color field and a semantic feature field [S1]. The DFF is distilled from pre-trained teacher networks like LSeg or DINO [S1]. This method allows for editing without the computational cost of iterative optimization-based methods [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
