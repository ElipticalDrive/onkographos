# NeuS Weight Function

Also called: NeuS Unbiased Density, NeuS density function, NeuS opaque density, SDF-based volume rendering weight

## Overview
The NeuS weight function is a volume rendering technique that derives an unbiased and occlusion-aware weight function $w(t)$ from a Signed Distance Function (SDF) $f(\mathbf{x})$ to reconstruct surfaces from multi-view images without explicit 3D supervision [S1, unofficial]. It solves the problem of geometric bias inherent in standard volume rendering formulations when applied to SDFs, ensuring that the volume-rendered surface aligns precisely with the geometric SDF zero-crossing [S2, unofficial]. Unlike its sibling [U-density Function](3695-u-density-function.md), which focuses on Unsigned Distance Fields, this method specifically addresses the asymmetry of SDFs to prevent the weight function from peaking before the actual surface intersection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$, where $\mathbf{o}$ is the camera center and $\mathbf{v}$ is the unit direction vector [S1, unofficial]. An SDF network $f(\mathbf{x})$ that outputs the signed distance at any point $\mathbf{x}$ [G]. A trainable inverse standard deviation parameter $s$ [S1, unofficial].
*   **Output:** The rendered color $C(\mathbf{o}, \mathbf{v})$ and the accumulated opacity/transmittance for training losses [S1, unofficial].

**Data Structures:**
*   **SDF Network:** A neural network mapping 3D coordinates to scalar SDF values [G].
*   **Sigmoid Function $\Phi_s$:** Defined as $\Phi_s(x) = (1 + e^{-sx})^{-1}$, where $s$ is a trainable parameter that approaches 0 as training converges [S2, unofficial].
*   **Sample Points:** A set of $n$ points $\{ \mathbf{p}_i = \mathbf{o} + t_i \mathbf{v} \mid i=1,\dots,n \}$ sampled along the ray [S1, unofficial].

**Formulas:**
1.  **Rendering Integral:**
    $$C(\mathbf{o}, \mathbf{v}) = \int_0^{+\infty} w(t) c(\mathbf{p}(t), \mathbf{v}) dt$$
    where $c(\mathbf{p}(t), \mathbf{v})$ is the color at point $\mathbf{p}(t)$ along viewing direction $\mathbf{v}$ [S1, unofficial].

2.  **Weight Function Definition:**
    $$w(t) = T(t)\rho(t)$$
    where $T(t)$ is the accumulated transmittance and $\rho(t)$ is the opaque density [S1, unofficial].

3.  **Accumulated Transmittance:**
    $$T(t) = \exp\left(-\int_0^t \rho(u) du\right)$$

4.  **Opaque Density $\rho(t)$:**
    The opaque density is derived to ensure the weight function is unbiased (peaks at $f(\mathbf{p}(t^*))=0$) and occlusion-aware (nearer points contribute more) [S1, unofficial].
    $$\rho(t) = \max \left( -\frac{d\Phi_s}{dt}(f(\mathbf{p}(t))), 0 \right)$$
    The term $-\frac{d\Phi_s}{dt}(f(\mathbf{p}(t)))$ represents the rate of change of the sigmoid SDF [S1, unofficial]. It is clipped to 0 to ensure $\rho(t)$ is non-negative, which occurs when the SDF value is increasing (i.e., the ray is leaving the surface) [S1, unofficial].

    *Derivation Note:* In the ideal case of a single plane intersection, the transmittance $T(t)$ equals $\Phi_s(f(\mathbf{p}(t)))$ [S1, unofficial]. Differentiating $\ln(T(t))$ yields the opaque density formula above [S1, unofficial].

5.  **Discrete Opacity $\alpha_i$:**
    For discrete sampling intervals $[t_i, t_{i+1}]$, the opacity $\alpha_i$ is computed as:
    $$\alpha_i = \max \left( \frac{\Phi_s(f(\mathbf{p}(t_i))) - \Phi_s(f(\mathbf{p}(t_{i+1})))}{\Phi_s(f(\mathbf{p}(t_i)))}, 0 \right)$$
    This is derived from $\alpha_i = 1 - \exp\left(-\int_{t_i}^{t_{i+1}} \rho(t) dt\right)$ [S1, unofficial].

6.  **Discrete Color Accumulation:**
    $$\hat{C} = \sum_{i=1}^n T_i \alpha_i c_i$$
    where $T_i = \prod_{j=1}^{i-1} (1-\alpha_j)$ is the discrete accumulated transmittance [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sigmoid_sdf(s: float, f_val: torch.Tensor) -> torch.Tensor:
    """Computes the sigmoid of the SDF value."""
    return 1.0 / (1.0 + torch.exp(-s * f_val))

def compute_neus_weight(s: float, f_vals: torch.Tensor, df_dt: torch.Tensor, colors: torch.Tensor) -> tuple:
    """
    Renders color using the NeuS weight function.
    Args:
        s: Trainable inverse standard deviation parameter.
        f_vals: SDF values f(p(t)) at n sample points along the ray.
        df_dt: Directional derivative of f along the ray direction v (df/dt).
        colors: Colors c(p(t)) at n sample points.
    Returns:
        rendered_color: Rendered color for the pixel.
        accumulated_opacity: Sum of weights (for mask loss).
    """
    # 1. Compute Phi_s
    phi_s = sigmoid_sdf(s, f_vals)
    
    # 2. Compute opaque density rho(t) = max(-d(Phi_s)/dt, 0)
    # Chain rule: d(Phi_s)/dt = d(Phi_s)/df * df/dt = s * Phi_s * (1 - Phi_s) * df/dt
    d_phi_s_d_f = s * phi_s * (1.0 - phi_s)
    d_phi_s_d_t = d_phi_s_d_f * df_dt
    rho = torch.clamp(-d_phi_s_d_t, min=0.0)
    
    # 3. Compute discrete opacity alpha_i
    # alpha_i = max((Phi_s(t_i) - Phi_s(t_{i+1})) / Phi_s(t_i), 0)
    phi_s_next = torch.cat([phi_s[1:], torch.zeros(1, device=phi_s.device)])
    denom = phi_s.clone()
    denom[denom == 0] = 1e-8
    alpha = torch.clamp((phi_s - phi_s_next) / denom, min=0.0)
    
    # 4. Compute transmittance T_i
    # T_i = product_{j=1}^{i-1} (1 - alpha_j), T_1 = 1
    one_minus_alpha = 1.0 - alpha
    cum_prod = torch.cumprod(one_minus_alpha, dim=0)
    T = torch.cat([torch.ones(1, device=phi_s.device), cum_prod[:-1]])
    
    # 5. Compute rendered color
    weights = T * alpha
    rendered_color = torch.sum(weights.unsqueeze(1) * colors, dim=0)
    accumulated_opacity = torch.sum(weights)
    
    return rendered_color, accumulated_opacity
```

**Parameters:**
*   $n$: Number of points sampled along each ray for discretization [S1, unofficial].
*   $s$: Trainable inverse standard deviation parameter. It is optimized during training and used in the sigmoid function $\Phi_s$ and the computation of the opaque density [S1, unofficial].
*   $\lambda$: Weight for the Eikonal regularization term [S1, unofficial].
*   $\beta$: Weight for the mask loss term [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bias in Naive Solution:** Using standard volume density $\sigma(t) = \phi_s(f(\mathbf{p}(t)))$ results in a weight function that peaks *before* the surface intersection, causing geometric bias [S1, unofficial].
*   **Occlusion Failure in Straightforward Unbiased Weight:** A simple normalized S-density weight $w(t) = \phi_s(f(\mathbf{p}(t))) / \int \phi_s(f(\mathbf{p}(u))) du$ is unbiased but not occlusion-aware. If a ray intersects multiple surfaces, it blends their colors equally regardless of depth [S1, unofficial].
*   **Negative Density:** The term $-\frac{d\Phi_s}{dt}$ can be negative when the SDF is increasing (ray leaving the surface). The $\max(\cdot, 0)$ clipping is essential to ensure non-negative density [S1, unofficial].

## Subtopics

### [NeUDF Hierarchical Importance Sampling](4863-neudf-hierarchical-importance-sampling.md)
A two-stage sampling strategy for Unsigned Distance Functions (UDFs) that decouples sampling distribution from rendering distribution to balance Eikonal loss gradients. It modifies the sampling density to handle UDF asymmetry, unlike the NeuS weight function which handles SDF asymmetry directly in the weight definition.

### [Naive UDF Solution (NeuS-based)](6015-naive-udf-solution-neus-based.md)
A direct extension of the NeuS weight function to UDFs. It uses a fixed, non-adaptive weight formulation derived from NeuS but suffers from geometric errors like redundant surfaces due to inherent bias in color rendering.

## Operating characteristics
*   **Time Complexity:** Dominated by the evaluation of the SDF network and its gradients for $n$ samples per ray [G]. Hierarchical sampling adds a coarse pass but reduces the number of fine samples needed for convergence [S1, unofficial].
*   **Memory Complexity:** Requires storing SDF values, gradients, colors, and intermediate transmittance/opaque values for $n$ samples per ray [G].
*   **Strengths:** Provides unbiased surface reconstruction by aligning the rendering weight peak with the SDF zero-crossing [S1, unofficial]. It is occlusion-aware, correctly handling multiple surface intersections [S1, unofficial].
*   **Limitations:** Relies on the first-order approximation of the SDF for unbiasedness [S2, unofficial]. Performance depends heavily on the quality of the SDF gradient (Eikonal regularization) [S1, unofficial].
*   **When to Use:** When reconstructing surfaces from multi-view images using SDFs and requiring high geometric accuracy without explicit 3D supervision [G].
*   **When Not to Use:** When using Unsigned Distance Fields (UDFs) directly without modification, as the SDF-specific derivation may not hold [G].

## Background
The NeuS weight function was introduced in the paper "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction" [S1, unofficial]. It addresses the limitations of previous neural implicit surface methods that suffered from geometric bias when using standard volume rendering formulations with SDFs [S2, unofficial]. The method builds on the idea that the weight function in volume rendering should be derived from the SDF to ensure the rendered surface matches the implicit surface defined by the SDF zero-level set [S1, unofficial].

## Sources
[S1] NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, https://doi.org/10.48550/arxiv.2106.10689, fetched 2026-10-26 (unofficial)
[S2] WACV 2023 Open Access Repository, https://openaccess.thecvf.com/content/WACV2023/html/Chen_Recovering_Fine_Details_for_Neural_Implicit_Surface_Reconstruction_WACV_2023_paper.html, fetched 2026-10-29 (unofficial)
