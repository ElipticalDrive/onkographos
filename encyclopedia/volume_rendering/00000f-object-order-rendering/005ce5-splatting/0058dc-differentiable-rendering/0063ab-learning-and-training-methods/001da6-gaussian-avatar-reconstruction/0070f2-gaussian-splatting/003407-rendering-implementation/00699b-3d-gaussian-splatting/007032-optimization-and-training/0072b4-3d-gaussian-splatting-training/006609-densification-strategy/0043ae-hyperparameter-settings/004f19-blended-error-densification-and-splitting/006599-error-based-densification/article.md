# Error-based Densification
Also called: Kernel Densification Strategy

## Overview
Error-based densification is a strategy for growing Gaussian primitives in 3D Gaussian Splatting that uses an auxiliary per-pixel error metric (e.g., Structural Similarity or $L_1$) to drive densification decisions, rather than relying on positional gradient magnitudes [S1, S2]. It solves the limitation of gradient-based methods, which are sensitive to error changes but blind to absolute error values in high-frequency areas, by redistributing per-pixel errors to primitives proportional to their rendering contribution [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Per-pixel error values $\mathcal{E}_\pi(\mathbf{u})$ computed between the rendered image and ground truth for camera $\pi$; current set of Gaussian primitives $\{\gamma_k\}$ [S1, S2].
*   **Output:** Updated densification scores $E_k$ for each primitive, used to determine if a primitive should be split or cloned [S1, S2].

**Data Structures:**
*   **Gaussian Primitive $\gamma_k$:** An additional scalar field $e_k$ is added to each primitive's data structure [S1].
    *   Field `e_k`: Scalar value, initialized to $0$ [S1]. This field is never updated during standard training steps; it is only updated via the auxiliary loss gradient [S1].
    *   Other fields (position, covariance, opacity, color, etc.) remain invariant with respect to this specific loss [S1].

**Numbered Steps:**
1.  **Render Scalar Field:** Enable rendering of the scalar field $e_k$ using a decoder $\Phi_{\text{ERR}}(\gamma_k, \mathbf{u}) := e_k$ [S1].
2.  **Compute Auxiliary Loss:** Calculate the auxiliary loss $L_\pi^{\text{aux}}$ for the current camera view $\pi$:
    $$L_\pi^{\text{aux}} := \sum_{\mathbf{u} \in \text{Pix}} \mathcal{W}[\mathcal{E}_\pi(\mathbf{u})] \mathcal{R}[\pi, \Phi_{\text{ERR}}](\mathbf{u})$$
    where $\mathcal{W}$ denotes gradient-detached error, and $\mathcal{R}[\pi, \Phi_{\text{ERR}}](\mathbf{u}) = \sum_{k=1}^K e_k \omega_k^\pi(\mathbf{u})$ is the rendered scalar value at pixel $\mathbf{u}$ [S1].
3.  **Backpropagate to Compute Primitive Error:** Compute the gradient of the auxiliary loss with respect to $e_k$. This gradient directly yields the per-primitive error for camera $\pi$:
    $$\frac{\partial L_\pi^{\text{aux}}}{\partial e_k} = \sum_{\mathbf{u} \in \text{Pix}} \mathcal{E}_\pi(\mathbf{u}) \omega_k^\pi(\mathbf{u}) = E_k^\pi$$
    where $\omega_k^\pi(\mathbf{u})$ is the contribution weight of primitive $\gamma_k$ to pixel $\mathbf{u}$ [S1].
4.  **Track Maximum Error:** Between two runs of the Adaptive Density Control (ADC) module, track the maximum error across all views $\pi \in \Pi$ seen:
    $$E_k := \max_{\pi \in \Pi} E_k^{\pi}$$
    This $E_k$ serves as the densification score [S1]. [S2] notes this accumulation occurs over a window $\Delta_{\text{error}}$.
5.  **Densification Decision:** Compare $E_k$ against a threshold $\gamma_{\text{error}}$. If $E_k > \gamma_{\text{error}}$, trigger densification (splitting or cloning) for primitive $\gamma_k$ [S1, S2].

**Formulas:**
*   **Per-Primitive Error per View:**
    $$E_k^\pi := \sum_{\mathbf{u} \in \text{Pix}} \mathcal{E}_\pi(\mathbf{u}) w_k^\pi(\mathbf{u})$$
    where $w_k^\pi(\mathbf{u})$ is the contribution weight (equivalent to $\omega_k^\pi(\mathbf{u})$ in the gradient derivation) [S1].
*   **Densification Score:**
    $$E_k := \max_{\pi \in \Pi} E_k^{\pi}$$
    where $\Pi$ is the set of views seen since the last ADC run [S1].
*   **Spatial Error Metric:** [S2] defines the per-pixel spatial error as $e(p) = \|c(p) - c_{GT}(p)\|_1$, while [S1] notes metrics like Structural Similarity are also used.

**Parameters:**
*   **Densification Threshold:** Grow Gaussians if $E_k > 0.1$ [S1]. [S2] denotes this as $\gamma_{\text{error}}$.
*   **Growth Limit:** Add up to $5\%$ of the current number of primitives at each densification step [S1]. [S2] denotes this as $\gamma_{\text{growth}}$.
*   **ADC Duration:** Keep the ADC process active for $27\text{k}$ iterations ($90\%$ of training) [S1].
*   **Accumulation Window:** Error is aggregated over a temporal window $\Delta_{\text{error}}$ [S2].
*   **Size Threshold:** Kernels are classified by spatial extent $(1 - \tau_s)$ against $\gamma_{\text{size}}$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_error_based_densification(primitives, views, gt_images, error_metric):
    """
    Computes the densification score E_k for each primitive based on error.
    primitives: list of primitive objects with attributes .id, .e_k, .opacity, .position, .covariance
    views: list of view parameters
    gt_images: dict mapping view.id to ground truth tensor
    error_metric: function computing per-pixel error (e.g., L1 or SSIM)
    """
    max_error = {k.id: 0.0 for k in primitives}
    
    for view in views:
        # 1. Render scalar field e_k via standard alpha blending
        rendered_scalar = torch.zeros_like(gt_images[view.id])
        weights = {}
        
        for k in primitives:
            # Compute contribution weight omega_k(u) for each pixel u in view
            # Standard Gaussian splatting weight computation
            diff = view.u_coords - k.position
            w = torch.exp(-0.5 * diff.T @ k.covariance_inv @ diff) * k.opacity
            weights[k.id] = w
            rendered_scalar += k.e_k * w
        
        # 2. Compute per-pixel error and detach gradients to prevent backprop through error metric
        per_pixel_error = error_metric(rendered_scalar, gt_images[view.id]).detach()
        
        # 3. Accumulate E_k^pi = sum_u (E_pi(u) * omega_k(u))
        for k in primitives:
            e_k_pi = torch.sum(per_pixel_error * weights[k.id])
            if e_k_pi > max_error[k.id]:
                max_error[k.id] = e_k_pi
                
    return max_error

def apply_densification(primitives, max_error, threshold=0.1, gamma_size=0.5, max_growth_ratio=0.05):
    """
    Triggers densification for primitives exceeding the error threshold.
    """
    candidates = [k for k in primitives if max_error[k.id] > threshold]
    max_new = int(len(primitives) * max_growth_ratio)
    candidates = candidates[:max_new]
    
    for k in candidates:
        # Spatial extent metric (1 - tau_s)
        spatial_extent = 1.0 - k.opacity
        if spatial_extent <= gamma_size:
            # Clone: copy primitive, reset e_k to 0, keep parameters invariant
            new_k = copy.deepcopy(k)
            new_k.id = len(primitives) + 1
            new_k.e_k = 0.0
            primitives.append(new_k)
        else:
            # Split: displace along principal axis via exponential map to respect manifold constraints
            split_k1, split_k2 = split_primitive_along_principal_axis(k)
            split_k1.e_k = 0.0
            split_k2.e_k = 0.0
            primitives.remove(k)
            primitives.extend([split_k1, split_k2])
```

**Edge Cases and Pitfalls:**
*   **Initialization:** $e_k$ must be initialized to $0$ for all primitives to ensure $L_\pi^{\text{aux}} = 0$ initially and parameters remain invariant except for $e_k$ [S1].
*   **Gradient Detachment:** The error metric $\mathcal{E}_\pi(\mathbf{u})$ must be gradient-detached ($\mathcal{W}$) to prevent backpropagation through the error calculation itself, ensuring only the scalar field $e_k$ is updated [S1].
*   **Opacity Bias:** When cloning, opacity preservation introduces bias by implicitly increasing the impact of the densified primitive, requiring opacity correction after cloning [S1].
*   **Manifold Constraints:** During splitting, new kernels must be displaced along their principal axis using the exponential map to respect manifold constraints [S2].
*   **Precision and Boundaries:** The threshold $\gamma_{\text{error}}$ and size cutoff $\gamma_{\text{size}}$ require careful tuning to avoid over-densification or missing high-frequency details [S1, S2].

## Operating characteristics
*   **Time Complexity:** Adds overhead proportional to the number of primitives and pixels for rendering the scalar field and computing the dot product for error redistribution [S1, S2].
*   **Memory Complexity:** Adds one scalar $e_k$ per primitive [S1].
*   **Strengths:** Directly targets areas of high reconstruction error, addressing the blindness of gradient-based methods to absolute error values in high-frequency regions [S1, S2]. Easier to set a threshold as it is expressed in terms of a known error metric [S1, S2].
*   **Limitations:** Requires access to ground truth for error computation during training. The method is sensitive to the choice of error metric (e.g., SSIM vs $L_1$) [S1, S2].
*   **When to Use:** When gradient-based densification fails to capture high-frequency details or absolute error magnitudes [S1, S2].
*   **When Not to Use:** If ground truth is unavailable or if computational budget is extremely tight, as it adds rendering and loss computation overhead [S1, S2].

## Background
Error-based densification was proposed in "Revising Densification in Gaussian Splatting" (2024) as an improvement over the original Adaptive Density Control (ADC) module in 3D Gaussian Splatting [S1]. It addresses the limitation that the original $\tau_k$ metric is sensitive to error changes but blind to absolute error values [S1]. The method was tested with training settings reproducing those of the original 3DGS and other baselines, using a densification threshold of $0.1$ and extending ADC to $27\text{k}$ iterations [S1]. A parallel formulation termed Kernel Densification Strategy appears in "Heat Kernel Textures the Geodesic Gaussians That Do Not Splat" (2024), which introduces an accumulation window $\Delta_{\text{error}}$, explicit size-based cloning/splitting thresholds, and exponential map displacement for splitting [S2].

## Sources
[S1] Revising Densification in Gaussian Splatting, https://doi.org/10.1007/978-3-031-73036-8_20, fetched 2026-10-26
[S2] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
