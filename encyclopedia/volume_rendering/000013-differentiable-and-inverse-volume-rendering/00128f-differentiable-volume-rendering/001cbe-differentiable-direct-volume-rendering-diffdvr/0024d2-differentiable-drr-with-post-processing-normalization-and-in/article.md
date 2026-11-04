# Differentiable DRR with Post-Processing Normalization and Inversion

Also called: DRR VJP normalization fix, SlangDiffDRRFunction gradient correction

## Overview
This method corrects the gradient computation in differentiable Digital Radiograph Reconstruction (DRR) pipelines where post-rendering normalization and inversion are applied outside the automatic differentiation graph. It solves the problem of anti-convergence in 2D/3D registration caused by omitted Jacobians for min-max normalization and clinical inversion, which flip gradient signs and scale magnitudes incorrectly [S1, unofficial]. It sets itself apart from standard differentiable volume rendering by explicitly handling the non-linear, pose-dependent boundary conditions introduced by global image statistics ($\min$ and $\max$) at the Python/CUDA interface [S1, unofficial].

## How it works
The method operates by computing the Vector-Jacobian Product (VJP) of the post-processing functions at the boundary between the differentiable renderer (e.g., Slang kernel) and the Python framework (e.g., PyTorch). This ensures that gradients flowing back to the raw Beer-Lambert intensities correctly account for the normalization and inversion transforms [S1, unofficial].

### Inputs and Outputs
**Inputs:**
1.  $r$: Raw rendered image tensor (shape $H \times W$), output of the differentiable DRR kernel.
2.  $g$: Upstream gradient tensor ($\partial L / \partial f$), where $f$ is the processed image.
3.  $\text{cfg}$: Configuration object containing:
    *   `normalize` (bool, default True): Whether min-max normalization is applied.
    *   `invert` (bool, default True): Whether clinical inversion is applied.
    *   $\varepsilon$ (float): Stability constant for normalization denominator.

**Outputs:**
1.  $\tilde{g}$: Corrected gradient tensor ($\partial L / \partial r$), ready to be uploaded to the backward kernel or used in the autograd engine.

### Data Structures
*   **Raw Image $r$**: Float32 tensor of shape $(H, W)$. Values represent Beer-Lambert intensity $I = i_0 \exp(-\int \mu ds)$.
*   **Gradient $g$**: Float32 tensor of shape $(H, W)$, same layout as $r$.
*   **Statistics**:
    *   $r_{\min} = \min(r)$: Scalar, minimum value in $r$.
    *   $r_{\max} = \max(r)$: Scalar, maximum value in $r$.
    *   $\Delta = r_{\max} - r_{\min}$: Scalar, dynamic range.

### Forward Pass Definition
The post-processing function $f(r)$ is defined as:
$$
f(r) = 1 - \frac{r - r_{\min}}{\Delta + \varepsilon}
$$
where $\Delta = \max(r) - \min(r)$ and $\varepsilon$ is a small stability constant (e.g., $10^{-6}$) to prevent division by zero [S1, unofficial].

### Backward Pass (VJP)
The goal is to compute $\nabla_r L = \frac{\partial L}{\partial r}$. By the chain rule:
$$
\nabla_r L = \left( \frac{\partial f}{\partial r} \right)^{\mathsf T} g
$$
The derivative $\frac{\partial f}{\partial r}$ has two components: the direct dependence of $f$ on $r$, and the indirect dependence through $r_{\min}$ and $r_{\max}$ [S1, unofficial].

1.  **Direct Term**:
    $$
    \frac{\partial f_i}{\partial r_i} = -\frac{1}{\Delta + \varepsilon}
    $$
    This term scales the gradient uniformly by $-\frac{1}{\Delta + \varepsilon}$.

2.  **Extremal Terms**:
    Since $r_{\min}$ and $r_{\max}$ depend on all pixels $r_j$, we must account for their gradients.
    $$
    \frac{\partial f_i}{\partial r_j} = \begin{cases} 
    \frac{1}{\Delta + \varepsilon} & \text{if } r_j = r_{\min} \text{ and } j \neq i \\
    -\frac{1}{\Delta + \varepsilon} & \text{if } r_j = r_{\max} \text{ and } j \neq i \\
    0 & \text{otherwise (for } j \neq i \text{)}
    \end{cases}
    $$
    For $j=i$, the direct term applies. If $r_i$ is the unique minimum or maximum, it contributes to both the direct term and the extremal term for itself. Specifically:
    *   If $r_i = r_{\min}$ (unique): $\frac{\partial f_i}{\partial r_i} = -\frac{1}{\Delta + \varepsilon} + \frac{1}{\Delta + \varepsilon} = 0$? No, careful.
    
    Let's derive $\frac{\partial f_i}{\partial r_j}$ rigorously.
    $$
    f_i = 1 - \frac{r_i - r_{\min}}{r_{\max} - r_{\min} + \varepsilon}
    $$
    Let $D = r_{\max} - r_{\min} + \varepsilon$.
    $$
    \frac{\partial f_i}{\partial r_j} = -\frac{1}{D} \delta_{ij} + \frac{r_i - r_{\min}}{D^2} \frac{\partial r_{\min}}{\partial r_j} - \frac{r_i - r_{\min}}{D^2} \frac{\partial r_{\max}}{\partial r_j} + \frac{1}{D} \frac{\partial r_{\min}}{\partial r_j}
    $$
    Wait, simpler approach using quotient rule on $u = r_i - r_{\min}$ and $v = D$:
    $$
    \frac{\partial f_i}{\partial r_j} = - \frac{ (\delta_{ij} - \frac{\partial r_{\min}}{\partial r_j}) D - (r_i - r_{\min}) (\frac{\partial r_{\max}}{\partial r_j} - \frac{\partial r_{\min}}{\partial r_j}) }{ D^2 }
    $$
    
    The derivatives of min/max are indicator functions:
    *   $\frac{\partial r_{\min}}{\partial r_j} = 1$ if $r_j = r_{\min}$, else 0.
    *   $\frac{\partial r_{\max}}{\partial r_j} = 1$ if $r_j = r_{\max}$, else 0.
    
    **Case 1: $j \neq i$**
    *   If $r_j = r_{\min}$ and $r_j \neq r_{\max}$:
        $$
        \frac{\partial f_i}{\partial r_j} = - \frac{ (-1) D - (r_i - r_{\min}) (-1) }{ D^2 } = \frac{ D + r_i - r_{\min} }{ D^2 } = \frac{ r_{\max} - r_{\min} + \varepsilon + r_i - r_{\min} }{ D^2 }
        $$
        This seems complex. Let's look at the source's simplified view. The source states:
        $$
        \nabla_\theta L = J_r^{\mathsf T}\left(\frac{\partial f}{\partial r}\right)^{\mathsf T}g = -\frac{1}{\Delta + \varepsilon}J_r^{\mathsf T}g + \text{terms through }\min(r)\text{ and }\max(r)
        $$
        The "terms through min/max" arise because $\min(r)$ and $\max(r)$ are pose-dependent. In the VJP at the image level, we compute $\tilde{g} = \frac{\partial f}{\partial r} g$.
        
        Let's implement the VJP directly as described in the fix:
        1.  Compute $r_{\min}, r_{\max}, \Delta$.
        2.  Compute mask for min pixels: $M_{\min} = (r == r_{\min})$.
        3.  Compute mask for max pixels: $M_{\max} = (r == r_{\max})$.
        4.  Handle ties: If multiple pixels share min/max, gradients should be shared. `torch.amin`/`torch.amax` behavior implies that if there are ties, the gradient is distributed or the derivative is subgradient. Standard practice in differentiable min/max is to distribute the gradient equally among tied extrema or use a soft approximation. The source notes: "Tied extrema at image boundaries require sharing gradients across tied pixels, matching `torch.amin` and `torch.amax` behavior." PyTorch's `amin`/`amax` backward pass typically returns 1 for all tied elements if using standard autograd, or requires custom handling. We assume standard autograd behavior where all tied pixels receive the full gradient contribution from the min/max operation if they are part of the set defining the extremum.
        
        Let $g_{\min} = \sum_{j: r_j = r_{\min}} g_j$ and $g_{\max} = \sum_{j: r_j = r_{\max}} g_j$.
        
        The gradient $\tilde{g}_i$ for pixel $i$ is:
        $$
        \tilde{g}_i = -\frac{g_i}{\Delta + \varepsilon} + \frac{g_{\min}}{\Delta + \varepsilon} \cdot \mathbb{I}(r_i = r_{\min}) - \frac{g_{\max}}{\Delta + \varepsilon} \cdot \mathbb{I}(r_i = r_{\max}) + \text{correction for } r_i \text{ being extremum}
        $$
        
        Actually, let's use the explicit derivative derived from $f_i = 1 - \frac{r_i - r_{\min}}{D}$.
        $$
        \frac{\partial f_i}{\partial r_j} = -\frac{\delta_{ij}}{D} + \frac{r_i - r_{\min}}{D^2} \frac{\partial r_{\min}}{\partial r_j} - \frac{r_i - r_{\min}}{D^2} \frac{\partial r_{\max}}{\partial r_j} + \frac{1}{D} \frac{\partial r_{\min}}{\partial r_j}
        $$
        Simplifying:
        $$
        \frac{\partial f_i}{\partial r_j} = -\frac{\delta_{ij}}{D} + \frac{1}{D} \frac{\partial r_{\min}}{\partial r_j} + \frac{r_i - r_{\min}}{D^2} \left( \frac{\partial r_{\min}}{\partial r_j} - \frac{\partial r_{\max}}{\partial r_j} \right)
        $$
        
        Summing over $j$ with weights $g_j$:
        $$
        \tilde{g}_i = \sum_j g_j \frac{\partial f_i}{\partial r_j} = -\frac{g_i}{D} + \frac{1}{D} \sum_{j: r_j=r_{\min}} g_j + \frac{r_i - r_{\min}}{D^2} \left( \sum_{j: r_j=r_{\min}} g_j - \sum_{j: r_j=r_{\max}} g_j \right)
        $$
        Let $S_{\min} = \sum_{j: r_j=r_{\min}} g_j$ and $S_{\max} = \sum_{j: r_j=r_{\max}} g_j$.
        $$
        \tilde{g}_i = -\frac{g_i}{D} + \frac{S_{\min}}{D} \mathbb{I}(r_i = r_{\min}) + \frac{r_i - r_{\min}}{D^2} (S_{\min} \mathbb{I}(r_i = r_{\min}) - S_{\max} \mathbb{I}(r_i = r_{\max}))
        $$
        
        If $r_i = r_{\min}$:
        $$
        \tilde{g}_i = -\frac{g_i}{D} + \frac{S_{\min}}{D} + \frac{0}{D^2} (S_{\min} - S_{\max}) = \frac{S_{\min} - g_i}{D}
        $$
        If $r_i = r_{\max}$ (and $r_{\max} \neq r_{\min}$):
        $$
        \tilde{g}_i = -\frac{g_i}{D} + 0 + \frac{r_{\max} - r_{\min}}{D^2} (0 - S_{\max}) = -\frac{g_i}{D} - \frac{\Delta S_{\max}}{D^2}
        $$
        If $r_i$ is neither:
        $$
        \tilde{g}_i = -\frac{g_i}{D}
        $$

    **Inversion Step**:
    If `invert` is True, the forward pass applies $f_{inv}(f) = 1 - f$ (or similar clinical inversion). The source notes: "Invert-only configuration yields exact negative gradient: $\partial f / \partial r = -1$".
    If both normalize and invert are applied, the total forward is $f_{total}(r) = 1 - f_{norm}(r)$.
    $$
    \frac{\partial f_{total}}{\partial r} = - \frac{\partial f_{norm}}{\partial r}
    $$
    So the corrected gradient $\tilde{g}_{total}$ is simply $-\tilde{g}_{norm}$, where $\tilde{g}_{norm}$ is the VJP computed above for the normalization step.

### Numbered Steps
1.  Receive raw image $r$ and upstream gradient $g$ (w.r.t processed image).
2.  Compute $r_{\min} = \min(r)$, $r_{\max} = \max(r)$, $\Delta = r_{\max} - r_{\min}$, $D = \Delta + \varepsilon$.
3.  Identify masks $M_{\min} = (r == r_{\min})$ and $M_{\max} = (r == r_{\max})$.
4.  Compute sums $S_{\min} = \sum_{j} g_j M_{\min}[j]$ and $S_{\max} = \sum_{j} g_j M_{\max}[j]$.
5.  Compute base gradient $\tilde{g}_{base} = -g / D$.
6.  Add min-term correction: $\tilde{g} = \tilde{g}_{base} + (S_{\min} / D) M_{\min}$.
7.  Add max-term correction: $\tilde{g} = \tilde{g} + ((r - r_{\min}) / D^2) (S_{\min} M_{\min} - S_{\max} M_{\max})$.
8.  If `invert` is True, negate $\tilde{g}$: $\tilde{g} = -\tilde{g}$.
9.  Return $\tilde{g}$.

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
        # If only invert is applied, derivative is -1
        grad_raw = -grad_output if do_invert else grad_output
        return grad_raw

    # Normalization VJP
    r_min = torch.amin(raw_image)
    r_max = torch.amax(raw_image)
    delta = r_max - r_min
    D = delta + eps
    
    # Masks for extrema
    # Note: torch.eq might have precision issues, but for integer-like or exact float comparisons in this context,
    # it matches the source's requirement to share gradients across tied pixels.
    mask_min = (raw_image == r_min)
    mask_max = (raw_image == r_max)
    
    # Sums of gradients at extrema
    s_min = torch.sum(grad_output * mask_min)
    s_max = torch.sum(grad_output * mask_max)
    
    # Base term: -g / D
    grad_raw = -grad_output / D
    
    # Min term correction: (S_min / D) * mask_min
    # Plus the term from the numerator derivative: (r_i - r_min) / D^2 * S_min * mask_min
    # Combined for min pixels: (S_min - g_i) / D ? 
    # Let's stick to the derived formula:
    # grad_raw = -g/D + (S_min/D)*mask_min + ((r - r_min)/D^2)*(S_min*mask_min - S_max*mask_max)
    
    term_min = (s_min / D) * mask_min
    term_max = (s_max / D) * mask_max
    
    # Numerator correction term
    numerator_correction = ((raw_image - r_min) / (D * D)) * (s_min * mask_min - s_max * mask_max)
    
    grad_raw = grad_raw + term_min + numerator_correction
    
    # Inversion
    if do_invert:
        grad_raw = -grad_raw
        
    return grad_raw
```

### Parameters
*   `normalize` (bool, default True): Applies min-max normalization.
*   `invert` (bool, default True): Inverts normalized image to clinical convention.
*   $\varepsilon$ (float): Stability constant added to normalization denominator. Typical value: $10^{-6}$.

### Edge Cases and Pitfalls
*   **Gradient Sign Flip**: Applying post-processing outside the autograd graph omits the Jacobian, flipping the gradient sign and scaling magnitude by $1/(\Delta + \varepsilon)$, causing anti-convergence [S1, unofficial].
*   **Tied Extrema**: If multiple pixels share the minimum or maximum value, gradients must be shared across them to match `torch.amin`/`torch.amax` behavior. Failure to do so causes incorrect gradient distribution [S1, unofficial].
*   **Pose-Dependent Extrema**: Switching extremal pixels with pose makes $\min(r)$ and $\max(r)$ pose-dependent, causing a piecewise smooth objective and decorrelating gradient components. The VJP terms through $\min$ and $\max$ are essential to handle this [S1, unofficial].
*   **Invert-Only Case**: If only inversion is applied, the gradient is exactly the negative of the upstream gradient [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(HW)$ for computing min/max, masks, and sums. The VJP application is also $O(HW)$. This is negligible compared to the DRR rendering cost.
*   **Memory Complexity**: $O(HW)$ for storing masks and intermediate tensors.
*   **Performance**: Correcting the gradient restores convergence. In 2D/3D registration on a chest CT, pose error decreased from growing to 137.6 mm/118.1 deg (buggy) to converging to 0.0005 mm/0.0009 deg (corrected) within 200 Adam iterations [S1, unofficial].
*   **Strengths**: Fixes critical convergence issues in differentiable registration workflows. Maintains compatibility with existing APIs by applying the fix at the boundary.
*   **Limitations**: Requires explicit handling of tied extrema. The pose-dependent min/max terms add complexity to the gradient computation.
*   **When to Use**: When using differentiable DRR with post-render normalization and inversion for tasks like 2D/3D registration.
*   **When Not to Use**: If raw Beer-Lambert intensities are used directly without post-processing, or if the post-processing is kept inside the autograd graph (e.g., using PyTorch operations on the GPU).

## Background
This method addresses a bug in the `i4h-sensor-simulation` package (specifically `SlangDiffDRRFunction`) where the backward pass ignored post-render normalization and inversion, leading to wrong pose gradients [S1, unofficial]. The issue was identified in GitHub issue #68, highlighting that the documented 2D/3D registration workflow anti-converged due to this omission [S1, unofficial]. The fix involves applying the VJP at the Python/CUDA boundary, ensuring that gradients are correctly propagated to the raw intensity space before being used by the differentiable renderer's backward kernel [S1, unofficial].

## Sources
[S1] fix(fluoro): backward pass ignores post-render normalisation and inversion, so pose gradients are wrong and the documented 2D/3D registration workflow anti-converges, https://github.com/isaac-for-healthcare/i4h-sensor-simulation/issues/68, fetched 2026-10-27 (unofficial)
