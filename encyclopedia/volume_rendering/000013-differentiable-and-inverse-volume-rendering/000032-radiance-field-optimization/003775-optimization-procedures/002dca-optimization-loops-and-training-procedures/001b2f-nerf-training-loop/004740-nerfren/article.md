# NeRFReN

Also called: Reflective NeRF

## Overview
NeRFReN addresses the failure of standard neural rendering to accurately represent reflective surfaces, such as glass, by separating the radiance field into transmitted and reflected components [S1]. It distinguishes itself from sibling methods like [NerfingMVS](9518-nerfingmvs.md) by explicitly modeling reflection artifacts as a distinct geometric entity rather than treating them as noise or occlusion, using bidirectional depth consistency to separate reflections from physical geometry [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images of a scene containing reflective surfaces; camera poses and intrinsics.
*   **Outputs:** A radiance field model comprising two sets of parameters: transmitted density $\sigma_t$ and color $c_t$, and reflected density $\sigma_r$ and color $c_r$ [S1].

**Data Structures:**
The model maintains two parallel neural networks (or MLPs) for each ray:
1.  **Transmitted Field:** Encodes geometry and color for the physical scene structure ($\sigma_t, c_t$).
2.  **Reflected Field:** Encodes geometry and color for the reflection artifacts ($\sigma_r, c_r$) [S1].

**Parameters:**
*   $\beta$: Reflection fraction, a scalar weight determining the contribution of the reflected component to the final pixel [S1].

**Formulas:**
The final pixel intensity $I$ is computed as a weighted sum of the transmitted intensity $I_t$ and reflected intensity $I_r$:
$$
I = I_t + \beta I_r \quad (1)
$$
where $\beta$ is determined by the geometry of the transmitted radiance field:
$$
\beta = \sum_i T_{\sigma_{it}} (1 - \exp(-\sigma_{it} \delta_i)) \alpha_i \quad (2)
$$
Definitions:
*   $T_{\sigma_{it}}$: Transmittance along the ray for the transmitted component, calculated as $\exp(-\sum_{j=1}^{i-1} \sigma_{it} \delta_j)$ [S1].
*   $\sigma_{it}$: Transmitted density at sample point $i$ [S1].
*   $\delta_i$: Step size between sample points [S1].
*   $\alpha_i$: A coefficient associated with the reflection at sample point $i$ [S1].

**Numbered Steps:**
1.  **Ray Sampling:** Sample points along each camera ray.
2.  **Field Evaluation:** Query the transmitted network for $\sigma_t$ and $c_t$, and the reflected network for $\sigma_r$ and $c_r$ at each sample point [S1].
3.  **Reflection Fraction Calculation:** Compute $\beta$ using Equation (2) based on the transmitted density $\sigma_t$ and step sizes $\delta_i$ [S1].
4.  **Volume Rendering:** Compute $I_t$ by standard volume rendering integration of $(\sigma_t, c_t)$ and $I_r$ by standard volume rendering integration of $(\sigma_r, c_r)$ [S1].
5.  **Composite Output:** Combine results using Equation (1) to produce the final pixel color [S1].
6.  **Loss Computation:**
    *   Apply standard photometric loss between rendered and ground truth images [S1].
    *   Apply depth smoothness loss $L_d$ to the transmitted field to encourage correct physical geometry [S1].
    *   Apply bidirectional depth consistency loss $L_{bdc}$ to the reflected field to ensure geometric plausibility of reflections [S1].
7.  **Optimization:** Update network parameters to minimize the total loss [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerfren_forward(ray_origin, ray_direction, transmitted_mlp, reflected_mlp, num_samples):
    """
    Renders a single ray using the NeRFReN model.
    """
    # 1. Sample points along the ray
    # Standard NeRF sampling strategy (e.g., uniform + stratified)
    t_vals = linspace(0, 1, num_samples)
    z_vals = near + t_vals * (far - near)
    points = ray_origin + ray_direction * z_vals[:, None]

    # 2. Query Transmitted Field
    sigma_t, c_t = transmitted_mlp(points)
    
    # 3. Query Reflected Field
    sigma_r, c_r = reflected_mlp(points)

    # 4. Compute Transmittance for Transmitted Field
    # delta_i is the distance between consecutive z_vals
    deltas = z_vals[1:] - z_vals[:-1]
    # Add epsilon to avoid division by zero or log(0) issues
    deltas = concat([deltas, 1e-10]) 
    
    # Transmittance T_i = exp(-sum_{j<i} sigma_j * delta_j)
    alpha_t = 1.0 - exp(-sigma_t * deltas)
    T_t = cumprod(1.0 - alpha_t + 1e-10)
    T_t = concat([1.0, T_t[:-1]]) # T_0 = 1

    # 5. Compute Reflection Fraction Beta
    # Beta = sum_i T_sigma_it * (1 - exp(-sigma_it * delta_i)) * alpha_i
    # Note: The source defines alpha_i as a coefficient. In standard NeRF, 
    # the weight for color is T_i * alpha_i. Here, the term (1 - exp(-sigma*delta)) 
    # is the opacity alpha. The source formula uses alpha_i as a separate coefficient.
    # Assuming alpha_i in Eq 2 refers to the opacity contribution or a specific learned weight.
    # Based on Eq 2 structure: beta = sum(T_t * (1 - exp(-sigma_t * deltas)) * alpha_coeffs)
    # If alpha_i is not explicitly defined as a separate network output in the snippet,
    # it may refer to the opacity term itself or a learned scalar. 
    # Given "alpha_i by Eq. (5)" in source, and Eq 5 in original NeRFReN paper 
    # typically relates to opacity, we assume alpha_i here represents the opacity 
    # contribution or a specific weighting factor. 
    # For implementation fidelity to the provided text:
    opacity_term = 1.0 - exp(-sigma_t * deltas)
    # alpha_i is a coefficient. If not provided as a separate tensor, it might be 1.0 
    # or derived from the reflected field. However, the text says "alpha_i by Eq. (5)".
    # In the absence of Eq 5 definition in the prompt, we treat alpha_i as a placeholder 
    # for a coefficient that must be defined by the specific implementation of the 
    # reflected field's contribution weight. 
    # *Correction*: In many reflective NeRF variants, the reflection weight is derived 
    # from the surface normal or density. The source explicitly links beta to transmitted 
    # geometry. We will assume alpha_i is a learned coefficient or 1 if not specified.
    # To be safe and self-contained, we define alpha_i as a learned parameter or 1.0 
    # if the source implies it's part of the standard opacity calculation.
    # However, Eq 2 sums T * (1-exp) * alpha. This looks like a weighted sum of opacities.
    # Let's assume alpha_i is 1.0 for the standard case unless specified otherwise, 
    # or that it represents the reflected field's opacity if coupled. 
    # Given the ambiguity of "alpha_i by Eq. (5)" without Eq 5, we note it as a coefficient.
    alpha_coeffs = ones_like(deltas) # Placeholder for alpha_i if not explicitly defined in snippet
    beta = sum(T_t * opacity_term * alpha_coeffs)

    # 6. Compute Transmitted Intensity It
    weights_t = T_t * alpha_t
    I_t = sum(weights_t * c_t)

    # 7. Compute Reflected Intensity Ir
    # Standard volume rendering for reflected component
    alpha_r = 1.0 - exp(-sigma_r * deltas)
    T_r = cumprod(1.0 - alpha_r + 1e-10)
    T_r = concat([1.0, T_r[:-1]])
    weights_r = T_r * alpha_r
    I_r = sum(weights_r * c_r)

    # 8. Composite
    I = I_t + beta * I_r
    
    return I, I_t, I_r, beta, sigma_t, sigma_r

def compute_losses(rendered_image, gt_image, sigma_t, sigma_r, depth_t, depth_r):
    """
    Computes the total loss for NeRFReN training.
    """
    # Standard photometric loss
    photometric_loss = mse(rendered_image, gt_image)
    
    # Depth smoothness loss for transmitted field
    # Encourages correct geometry
    depth_smoothness_loss = compute_smoothness_loss(depth_t)
    
    # Bidirectional depth consistency loss for reflected field
    # Separates reflection artifacts from physical entities
    bidirectional_consistency_loss = compute_bidirectional_consistency(depth_t, depth_r)
    
    total_loss = photometric_loss + depth_smoothness_loss + bidirectional_consistency_loss
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Complex Interlacing:** The method cannot distinguish between separation when physical entities and reflection artifacts are complexly interlaced [S2].
*   **Reflection Fraction Calculation:** The calculation of $\beta$ relies heavily on the accuracy of the transmitted density $\sigma_t$. Errors in the transmitted geometry will propagate to the reflection weighting [S1].

## Operating characteristics
*   **Time Complexity:** Training time is increased compared to standard NeRF due to the dual-network architecture and additional loss computations ($L_d$ and $L_{bdc}$) [S1].
*   **Memory Complexity:** Memory usage is approximately doubled compared to standard NeRF as it requires storing parameters for both transmitted and reflected fields [S1].
*   **Performance:** Outperforms baseline NeRF and NerfingMVS on scenes with reflective surfaces (e.g., the RFFR dataset) [S1].
*   **Strengths:** Enables scene editing via reflection removal and substitution; effectively handles common failure cases like glasses [S1].
*   **Limitations:** Fails to generalize when occlusion objects are light-damaged transparent glass bodies or when entities and artifacts are complexly interlaced [S2].

## Background
NeRFReN was introduced in November 2021 [S1]. It was developed to address the specific problem of reflective surfaces in neural rendering view synthesis, a common failure case for standard approaches [S1]. The method was evaluated on the RFFR dataset and demonstrated the ability to support scene editing tasks such as reflection removal and substitution [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
