# Differentiable DRR with Post-Processing Normalization and Inversion

Also called: DRR VJP normalization fix, SlangDiffDRRFunction gradient correction

## Overview
This method corrects the gradient computation in differentiable Digital Radiograph Reconstruction (DRR) pipelines where post-rendering normalization and inversion are applied outside the automatic differentiation graph [S1, unofficial]. It solves the problem of anti-convergence in 2D/3D registration caused by omitted Jacobians for min-max normalization and clinical inversion, which flip gradient signs and scale magnitudes incorrectly [S1, unofficial]. It sets itself apart from standard differentiable volume rendering by explicitly handling the non-linear, pose-dependent boundary conditions introduced by global image statistics ($\min$ and $\max$) at the Python/CUDA interface [S1, unofficial].

## How it works
The method computes the Vector-Jacobian Product (VJP) of the post-processing functions at the boundary between the differentiable renderer and the Python framework [S1, unofficial]. This ensures that gradients flowing back to the raw Beer-Lambert intensities correctly account for the normalization and inversion transforms [S1, unofficial].

### Inputs and Outputs
**Inputs:**
1. $r$: Raw rendered image tensor of shape $(H, W)$, output of the differentiable DRR kernel [S1, unofficial].
2. $g$: Upstream gradient tensor of shape $(H, W)$ representing $\partial L / \partial f$, where $f$ is the processed image [S1, unofficial].
3. $\text{cfg}$: Configuration object containing `normalize` (bool, default True), `invert` (bool, default True), and $\varepsilon$ (float stability constant) [S1, unofficial].

**Outputs:**
1. $\tilde{g}$: Corrected gradient tensor of shape $(H, W)$ representing $\partial L / \partial r$, ready for the backward kernel [S1, unofficial].

### Data Structures
* **Raw Image $r$**: Float32 tensor of shape $(H, W)$ representing Beer-Lambert intensity $I = i_0 \exp(-\int \mu ds)$ [S1, unofficial].
* **Gradient $g$**: Float32 tensor of shape $(H, W)$ with the same layout as $r$ [S1, unofficial].
* **Statistics**: $r_{\min} = \min(r)$, $r_{\max} = \max(r)$, $\Delta = r_{\max} - r_{\min}$, and $D = \Delta + \varepsilon$ [S1, unofficial].

### Forward Pass Definition
The post-processing function $f(r)$ is defined as:
$$
f(r) = 1 - \frac{r - r_{\min}}{D}
$$
where $D = \max(r) - \min(r) + \varepsilon$ and $\varepsilon$ is a small stability constant to prevent division by zero [S1, unofficial].

### Backward Pass (VJP)
The goal is to compute $\nabla_r L = \frac{\partial L}{\partial r}$ using the chain rule $\nabla_r L = \left( \frac{\partial f}{\partial r} \right)^{\mathsf T} g$ [S1, unofficial]. The derivative $\frac{\partial f_i}{\partial r_j}$ contains a direct term and indirect terms through $r_{\min}$ and $r_{\max}$ [S1, unofficial]. Applying the quotient rule yields:
$$
\frac{\partial f_i}{\partial r_j} = -\frac{\delta_{ij}}{D} - \frac{1}{D}\frac{\partial r_{\min}}{\partial r_j} + \frac{r_i - r_{\min}}{D^2}\left(\frac{\partial r_{\max}}{\partial r_j} - \frac{\partial r_{\min}}{\partial r_j}\right)
$$
The derivatives of the extrema are indicator functions: $\frac{\partial r_{\min}}{\partial r_j} = \mathbb{I}(r_j = r_{\min})$ and $\frac{\partial r_{\max}}{\partial r_j} = \mathbb{I}(r_j = r_{\max})$ [G]. Summing over $j$ with weights $g_j$ gives the corrected gradient for pixel $i$:
$$
\tilde{g}_i = -\frac{g_i}{D} - \frac{S_{\min}}{D}\mathbb{I}(r_i = r_{\min}) + \frac{r_i - r_{\min}}{D^2}\left(S_{\max}\mathbb{I}(r_i = r_{\max}) - S_{\min}\mathbb{I}(r_i = r_{\min})\right)
$$
where $S_{\min} = \sum_{j: r_j = r_{\min}} g_j$ and $S_{\max} = \sum_{j: r_j = r_{\max}} g_j$ [S1, unofficial]. If `invert` is True, the total forward pass applies $1 - f(r)$, so the final gradient is negated: $\tilde{g}_{\text{total}} = -\tilde{g}$ [S1, unofficial].

### Numbered Steps
1. Receive raw image $r$ and upstream gradient $g$ [S1, unofficial].
2. Compute $r_{\min} = \min(r)$, $r_{\max} = \max(r)$, $\Delta = r_{\max} - r_{\min}$, and $D = \Delta + \varepsilon$ [S1, unofficial].
3. Identify masks $M_{\min} = (r == r_{\min})$ and $M_{\max} = (r == r_{\max})$ [S1, unofficial].
4. Compute sums $S_{\min} = \sum_j g_j M_{\min}[j]$ and $S_{\max} = \sum_j g_j M_{\max}[j]$ [S1, unofficial].
5. Compute base gradient $\tilde{g}_{\text{base}} = -g / D$ [S1, unofficial].
6. Add min-term correction: $\tilde{g} = \tilde{g}_{\text{base}} - (S_{\min} / D) M_{\min}$ [S1, unofficial].
7. Add max-term correction: $\tilde{g} = \tilde{g} + ((r - r_{\min}) / D^2) (S_{\max} M_{\max} - S_{\min} M_{\min})$ [S1, unofficial].
8. If `invert` is True, negate $\tilde{g}$: $\tilde{g} = -\tilde{g}$ [S1, unofficial].
9. Return $\tilde{g}$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def postprocess_vjp(raw_image: torch.Tensor, grad_output: torch.Tensor, cfg: dict) -> torch.Tensor:
    """
    Computes the VJP of the post-processing (normalize + invert) for DRR.
    
    Args:
        raw_image: Tensor of shape (H, W), raw Beer-Lambert intensities.
        grad_output: Tensor of shape (H, W), gradient w.r.t processed image.
        cfg: Dict with 'normalize' (bool), 'invert' (bool), 'eps' (float).
        
    Returns:
        Tensor of shape (H, W), gradient w.r.t raw_image.
    """
    eps = cfg.get('eps', 1e-6)
    do_normalize = cfg.get('normalize', True)
    do_invert = cfg.get('invert', True)
    
    if not do_normalize:
        return -grad_output if do_invert else grad_output

    r_min = torch.amin(raw_image)
    r_max = torch.amax(raw_image)
    D = (r_max - r_min) + eps
    
    mask_min = (raw_image == r_min)
    mask_max = (raw_image == r_max)
    
    s_min = torch.sum(grad_output * mask_min)
    s_max = torch.sum(grad_output * mask_max)
    
    grad_raw = -grad_output / D
    grad_raw -= (s_min / D) * mask_min
    grad_raw += ((raw_image - r_min) / (D * D)) * (s_max * mask_max - s_min * mask_min)
    
    if do_invert:
        grad_raw = -grad_raw
        
    return grad_raw
```

### Parameters
* `normalize` (bool, default True): Applies min-max normalization to the raw image [S1, unofficial].
* `invert` (bool, default True): Inverts the normalized image to clinical convention [S1, unofficial].
* $\varepsilon$ (float): Stability constant added to the normalization denominator to prevent division by zero [S1, unofficial].

### Edge Cases and Pitfalls
* **Gradient Sign Flip**: Applying post-processing outside the autograd graph omits its Jacobian, flipping the gradient sign and scaling magnitude by $1/(\Delta + \varepsilon)$, which causes anti-convergence [S1, unofficial].
* **Tied Extrema**: Multiple pixels sharing the minimum or maximum value require sharing gradients across them to match `torch.amin` and `torch.amax` behavior [S1, unofficial].
* **Pose-Dependent Extrema**: Switching extremal pixels with pose makes $\min(r)$ and $\max(r)$ pose-dependent, causing a piecewise smooth objective and decorrelating gradient components [S1, unofficial].
* **Invert-Only Case**: If only inversion is applied, the returned gradient is exactly the negative of the true one [S1, unofficial].

## Operating characteristics
* **Time Complexity**: $O(HW)$ for computing extrema, masks, sums, and applying the VJP [S1, unofficial].
* **Memory Complexity**: $O(HW)$ for storing masks and intermediate tensors [S1, unofficial].
* **Performance**: Correcting the gradient restores convergence, reducing pose error from growing to 137.6 mm/118.1 deg to converging to 0.0005 mm/0.0009 deg within 200 Adam iterations on a chest CT [S1, unofficial].
* **Strengths**: Fixes critical convergence issues in differentiable registration workflows while preserving existing API outputs [S1, unofficial].
* **Limitations**: Requires explicit handling of tied extrema and pose-dependent min/max terms [S1, unofficial].
* **When to Use**: When using differentiable DRR with post-render normalization and inversion for 2D/3D registration workflows [S1, unofficial].
* **When Not to Use**: When raw Beer-Lambert intensities are used directly without post-processing, or when post-processing is kept inside the autograd graph [S1, unofficial].

## Background
This method addresses a bug in the `i4h-sensor-simulation` package where the backward pass ignored post-render normalization and inversion, leading to wrong pose gradients [S1, unofficial]. The omission was identified in the package's issue tracker, highlighting that the documented 2D/3D registration workflow anti-converged due to this omission [S1, unofficial]. The fix applies the VJP at the Python/CUDA boundary, ensuring that gradients are correctly propagated to the raw intensity space before being used by the differentiable renderer's backward kernel [S1, unofficial]. It is used in 2D/3D registration workflows for medical imaging, specifically for differentiable DRR generation with clinical normalization conventions [S1, unofficial].

## Sources
[S1] fix(fluoro): backward pass ignores post-render normalisation and inversion, so pose gradients are wrong and the documented 2D/3D registration workflow anti-converges, https://github.com/isaac-for-healthcare/i4h-sensor-simulation/issues/68, fetched 2026-10-27 (unofficial)
