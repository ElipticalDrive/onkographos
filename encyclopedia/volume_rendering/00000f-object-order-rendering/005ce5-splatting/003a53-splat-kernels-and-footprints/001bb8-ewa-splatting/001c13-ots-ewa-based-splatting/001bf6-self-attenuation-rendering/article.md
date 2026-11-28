# Self-Attenuation Rendering

Also called: EWA-Based Splatting with Self-Attenuation, Exact Self-Attenuation, OTS+SAtn, Self-Attenuation in 3DGS, Self-Attenuation Volumetric Splatting

## Overview
Self-Attenuation Rendering is a volumetric splatting technique that accurately accounts for the exponential extinction of a Gaussian primitive along the viewing ray, rather than ignoring it or using a first-order Taylor approximation [S1]. It solves the problem of visual inaccuracies and training instability caused by the standard approximation $1-g(\mathbf{p})$ for attenuation, which fails when the extinction coefficient $g(\mathbf{p})$ exceeds 1 [S1]. Unlike standard EWA Splatting and 3DGS, which approximate the attenuation term, this method uses the exact exponential form $1-e^{-f(\mathbf{p})}$, remaining valid for any $f(\mathbf{p}) \in [0, \infty)$ without requiring clamping [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians, each defined by mean $\mu_i$, covariance $\Sigma_i$, opacity parameter $\theta_i$ (or mass), and color $c_i(\mathcal{R})$ [S1]. For each pixel $\mathbf{p}$, the method requires the 2D projected Gaussian parameters derived from the 3D Gaussians (as defined in [OTS (EWA-Based Splatting)](1181-ots-ewa-based-splatting.md)) [G].
*   **Outputs:** The final pixel intensity $I(\mathbf{p})$ [S1].

**Data Structures:**
*   **Gaussian List:** A list of Gaussians sorted by depth (view-space $z$) for each pixel [S1]. The sorting order is critical for the front-to-back accumulation [S1].
*   **2D Extinction $f_i(\mathbf{p})$:** The integrated extinction coefficient of Gaussian $i$ at pixel $\mathbf{p}$, derived from the 3D Gaussian's mass and projection [S1]. In the context of OTS, this is computed using the 2D Gaussian extinction function defined in the parent article [S1].

**Formulas:**
The rendering equation for a single Gaussian $i$ at pixel $\mathbf{p}$ is:
$$ I_i(\mathbf{p}) = c_i(\mathcal{R}) \left( 1 - e^{-f_i(\mathbf{p})} \right) $$
where $c_i(\mathcal{R})$ is the color of Gaussian $i$ and $f_i(\mathbf{p})$ is the 2D Gaussian extinction [S1].

For multiple non-overlapping Gaussians sorted by depth ($i=0$ to $N-1$, where $0$ is closest to the camera), the total intensity is:
$$ I(\mathbf{p}) = \sum_{i=0}^{N-1} c_i(\mathcal{R}) \left(1 - e^{-f_i(\mathbf{p})}\right) \prod_{j=0}^{i-1} \left(e^{-f_j(\mathbf{p})}\right) + c_b \prod_{i=0}^{N-1} \left(e^{-f_i(\mathbf{p})}\right) $$
where $c_b$ is the background color [S1].

**Parameters:**
*   **Activation Function:** The opacity parameter $\theta$ (or mass) is activated using **softplus** with $\beta = 2$ [S1]. This is required because the self-attenuation model does not use the Taylor approximation for the exponential term; thus, a sigmoid activation would prevent Gaussians from becoming fully opaque from all directions [S1].
*   **Extinction Coefficient:** $f_i(\mathbf{p})$ is computed based on the 2D Gaussian projection. The exact formula for $f_i(\mathbf{p})$ relies on the OTS definition of 2D extinction [S1].

**Steps:**
1.  **Project and Sort:** For each pixel, project the 3D Gaussians to 2D. Sort the visible Gaussians by depth (front-to-back) [S1].
2.  **Compute Extinction:** For each Gaussian $i$ in the sorted list, compute the 2D extinction coefficient $f_i(\mathbf{p})$ at the pixel location [S1].
3.  **Accumulate Transmittance:** Initialize accumulated transmittance $T = 1.0$ [S1].
4.  **Render Loop:** Iterate through the sorted Gaussians $i = 0$ to $N-1$:
    a. Compute the self-attenuation factor: $A_i = 1 - e^{-f_i(\mathbf{p})}$ [S1].
    b. Add contribution to color: $C_{out} \leftarrow C_{out} + c_i(\mathcal{R}) \cdot A_i \cdot T$ [S1].
    c. Update transmittance: $T \leftarrow T \cdot e^{-f_i(\mathbf{p})}$ [S1].
5.  **Background:** Add background contribution: $C_{out} \leftarrow C_{out} + c_b \cdot T$ [S1].
6.  **Output:** Return $C_{out}$ as the pixel color [S1].

**Pseudocode:**
```python
import math

def render_self_attenuation(gaussians_sorted_by_depth, background_color):
    """
    Renders a pixel using self-attenuation splatting.
    gaussians_sorted_by_depth: List of Gaussian objects sorted front-to-back.
    Each Gaussian has attributes: color (c), extinction (f).
    """
    accumulated_color = 0.0
    accumulated_transmittance = 1.0
    
    for gaussian in gaussians_sorted_by_depth:
        f_i = gaussian.compute_extinction_at_pixel()
        
        # Self-attenuation term: 1 - exp(-f_i)
        self_attenuation = 1.0 - math.exp(-f_i)
        
        # Add contribution: color * self_attenuation * accumulated_transmittance
        accumulated_color += gaussian.color * self_attenuation * accumulated_transmittance
        
        # Update transmittance: multiply by exp(-f_i)
        accumulated_transmittance *= math.exp(-f_i)
        
        # Optimization: If transmittance is negligible, break early
        if accumulated_transmittance < 1e-6:
            break
            
    # Add background color multiplied by final transmittance
    accumulated_color += background_color * accumulated_transmittance
    
    return accumulated_color
```

**Backward Pass (Gradient Computation):**
*   Standard 3DGS uses a back-to-front traversal for the backward pass to recover gradients efficiently by storing the last Gaussian index and remaining opacity [S1].
*   This approach causes artifacts with self-attenuation because the linear falloff of the Taylor approximation is replaced with a true exponential falloff [S1].
*   **Solution:** Compute the gradient in the backward pass using a **front-to-back** traversal strategy [S1]. This works because volumetric rendering (and its gradients) can be performed in either direction [S1]. This requires an extensive rewrite of the gradient computation kernel compared to standard 3DGS [S1].

## Subtopics

### Self-Attenuation Backward Pass
A variant of the gradient computation for self-attenuation splatting. Standard 3DGS uses front-to-back forward blending and back-to-front backward gradient computation, storing the index of the last Gaussian and remaining opacity to optimize memory and computation [S1]. However, this back-to-front backward pass causes heavy artifacts when self-attenuation is used because the linear falloff of the Taylor approximation is replaced with a true exponential falloff [S1]. To fix this, the gradient is computed in the backward pass using a front-to-back strategy, leveraging the fact that volumetric rendering (and thus gradients) can be performed in either direction [S1]. This requires storing intermediate values differently than the standard 3DGS backward pass [S1].

## Operating characteristics
*   **Accuracy:** Provides visually accurate results by correctly modeling self-attenuation, even when Gaussians do not overlap [S1]. It avoids the "popping" and visibility artifacts associated with approximations when $g(\mathbf{p}) > 1$ [S1].
*   **Stability:** Removes the need for clamping $f_i(\mathbf{p})$ to $[0,1)$, which was necessary in standard EWA/3DGS to prevent invalid attenuation values [S1]. This ensures gradients remain essential for training and avoids sharp, non-Gaussian falloffs [S1].
*   **When to use:** Use when high visual fidelity is required, especially for large primitives or scenes where Gaussians have high mass/opacity, causing $f(\mathbf{p})$ to exceed 1 [S1]. It is particularly important for training stability in 3DGS when using unrestricted opacity parameters [S1].

## Background
Self-Attenuation Rendering was introduced in the paper "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" [S1]. The paper highlights that both EWA splatting and 3DGS ignore how a Gaussian's extinction affects its own appearance, a phenomenon termed "self-attenuation" [S1]. The authors demonstrate that this affects the visual result even when Gaussians do not overlap [S1]. The method extends the OTS (EWA-Based Splatting) framework by replacing the first-order Taylor approximation of the attenuation term with the exact exponential form [S1]. The implementation requires changes to both the forward rendering kernel and the backward gradient computation kernel, specifically switching the backward pass to front-to-back traversal [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
