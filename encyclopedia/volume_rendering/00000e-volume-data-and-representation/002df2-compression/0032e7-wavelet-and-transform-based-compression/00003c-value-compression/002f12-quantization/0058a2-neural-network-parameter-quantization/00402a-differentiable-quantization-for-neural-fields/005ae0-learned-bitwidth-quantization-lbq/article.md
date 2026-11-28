# Learned Bitwidth Quantization (LBQ)
## Overview
Learned Bitwidth Quantization (LBQ) is a differentiable quantization framework for neural radiance fields that treats integer bitwidth as a continuous, learnable variable ("soft bitwidth") rather than a fixed hyperparameter [S1]. It solves the accuracy-efficiency trade-off by automatically aligning model complexity with scene intricacy, eliminating the need for trial-and-error bitwidth selection required by traditional Quantization-Aware Training (QAT) [S1]. Unlike fixed-bitwidth QAT methods like LSQ, LBQ allows bitwidth to adapt per-component during training via gradient descent [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A radiance field model with parameters $\Theta$ (multi-resolution feature table) and $\Phi$ (MLP weights), and training data consisting of rays $\mathbf{r}$ with origin $o$ and direction $\mathbf{d}$ [S1].
*   **Output:** A quantized model where each component has an optimized integer bitwidth $B$, range scale $r_v$, and offset $v_{max}$, producing rendered images $\mathbf{C}(\mathbf{r})$ with minimal loss compared to full precision [S1].

**Data Structures:**
The radiance field maps 5D input $(\mathbf{x}, \mathbf{d})$ to opacity $\sigma$ and color $\mathbf{c}$ [S1]:
$$ F: (\mathbf{x}, \mathbf{d}) \xrightarrow{} (\sigma , \mathbf{c}) $$
where $\mathbf{x}$ is 3D location and $\mathbf{d}$ is 2D view direction [S1].
1.  **Encoding:** Location $\mathbf{x}$ is encoded as $\mathbf{y}_x = \text{enc}(\mathbf{x}, \Theta)$ [S1]. View direction $\mathbf{d}$ is encoded as $\mathbf{y}_d$ using spherical harmonics [S1].
2.  **MLP:** The encoded vector $\mathbf{y} = (\mathbf{y}_x, \mathbf{y}_d)$ is fed to an MLP to produce $(\sigma , \mathbf{c}) = m(\mathbf{y}, \mathbf{\Phi})$ [S1].
3.  **Rendering:** Expected color along ray $\mathbf{r}$ is calculated via volume rendering:
    $$ \mathbf{C}(\mathbf{r}) = \int ^{t_\text {f}}_{t_\text {n}}T(t) \sigma ( \mathbf{r}(t) ) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) dt $$
    where $T(t)= \exp (-\int ^t_{t_{\text {n}}} \sigma ( \mathbf{r}(s) ) ds )$ and $\mathbf{r}(t) = o + t\mathbf{d}$ [S1].

**Quantization Parameters:**
Each component (weights, activations) is associated with:
*   $b$: Soft bitwidth (float) [S1].
*   $B$: Integer bitwidth, $B = \lfloor b \rceil$ (round to nearest integer) [S1].
*   $r_v$: Range scale for input data $v$ (trainable) [S1].
*   $v_{max}$: Offset for quantization range (trainable) [S1].
*   $r_q$: Quantization range scale, $r_q = 2^B - 1$ [S1].
*   $Z$: Zero-point offset, $Z = \lfloor q_{\text {max}} - v_{\text {max}}/s \rceil$ [S1].
*   $s$: Step size, $s = r_v / r_q$ [S1].
*   $q_{\min}, q_{\max}$: Quantization bounds determined by the component's scheme (signed/unsigned) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def clamp(x, a, c):
    return max(a, min(c, x))

def lbq_quantize(v, b, rv, vmax, q_min, q_max):
    """
    Differentiable fake quantization for LBQ.
    Returns quantized value and gradients w.r.t b, rv, vmax.
    """
    # Straight-Through Estimator: value is rounded, gradient is 1
    B = round(b)
    rq = 2**B - 1
    s = rv / rq
    Z = math.floor(q_max - vmax / s)

    # Quantize and de-quantize
    V_hat = round(v / s) + Z
    V_clamped = clamp(V_hat, q_min, q_max)
    v_hat = s * (V_clamped - Z)

    # Compute gradients using analytical derivatives from Table 1 [S1]
    v_min = s * (q_min - Z)
    dv_hat_d_b = 0.0
    dv_hat_d_vmax = 0.0
    dv_hat_d_rv = 0.0

    if v_min <= v <= vmax:
        dv_hat_d_rv = (s * round(v / s) - v) / rv
        dv_hat_d_b = (v - s * round(v / s)) * (2**B * math.log(2)) / rq
        dv_hat_d_vmax = 0.0
    elif v > vmax:
        dv_hat_d_rv = 1.0 - vmax / rv - Z / rq
        dv_hat_d_b = 0.0
        dv_hat_d_vmax = (vmax - rv + s * Z) * (2**B * math.log(2)) / rq
    else: # v < v_min
        dv_hat_d_rv = -vmax / rv - Z / rq
        dv_hat_d_b = 0.0
        dv_hat_d_vmax = 1.0

    return v_hat, dv_hat_d_b, dv_hat_d_rv, dv_hat_d_vmax

def train_lbq_step(model, rays, targets, lr):
    """
    One training step for LBQ.
    model: Radiance field with parameters Theta, Phi, and quantization params per component.
    rays: List of rays with origin o and direction d.
    targets: Ground truth colors.
    lr: Learning rate.
    """
    rendered_colors = []
    for ray in rays:
        o, d = ray.origin, ray.direction
        # Discrete approximation of volume rendering integral [S1]
        t_vals = [ray.t_n + i * (ray.t_f - ray.t_n) / ray.num_samples for i in range(ray.num_samples)]
        T = 1.0
        C_ray = [0.0, 0.0, 0.0]
        for t in t_vals:
            x = o + t * d
            y_x = model.encode(x, model.Theta)
            y_d = model.encode_spherical_harmonics(d)
            y = y_x + y_d
            # Apply fake quantizers to activations/weights during forward pass
            y_quant, grad_b, grad_rv, grad_vmax = lbq_quantize(y, model.b, model.rv, model.vmax, model.q_min, model.q_max)
            sigma, c = model.MLP(y_quant, model.Phi)
            alpha = 1.0 - math.exp(-sigma * (ray.t_f - ray.t_n) / ray.num_samples)
            T_term = T
            C_ray = [C_ray[i] + T_term * alpha * c[i] for i in range(3)]
            T *= (1.0 - alpha)
        rendered_colors.append(C_ray)

    # Compute mean squared error loss [S1]
    loss = sum(sum((c[i] - t[i])**2 for i in range(3)) for c, t in zip(rendered_colors, targets)) / (3 * len(targets))

    # Gradient descent update for all parameters [S1]
    for param in model.parameters():
        param -= lr * param.grad
```

**Steps:**
1.  **Insert Fake Quantizers:** Place fake quantizers in encoding and MLP components [S1].
2.  **Initialize Parameters:** Initialize all components to 8-bit, except exponential activation to 32-bit [S1]. Initialize quantization parameters using a simple PTQ method [S1].
3.  **Simulate Quantization:** Perform quantize and de-quantize on data $v$ using the fake quantizer function [S1].
4.  **Compute Loss:** Calculate rendering loss $\mathcal{L}^{\text{NeRF}}$ [S1].
5.  **Update Parameters:** Use gradient descent to update $b$, $r_v$, $v_{max}$, and model weights. The gradients for $b$ and $r_v$ depend on the quantization error term $(v - s \cdot \lfloor v/s \rceil)$ [S1]. $v_{max}$ is calibrated upon overflow [S1].

**Parameters:**
*   $b$: Soft bitwidth. Initialized to 8.0 for most components, 32.0 for exponential activation [S1].
*   $r_v$: Range scale. Initialized via PTQ [S1].
*   $v_{max}$: Offset. Initialized via PTQ [S1].
*   Bitwidth Range: Constrained to $[2, 32]$ [S1]. Binary (1-bit) is excluded as it yields nonsensical results [S1].

**Edge Cases and Pitfalls:**
*   **Binary Quantization:** 1-bit quantization produces nonsensical rendering and is excluded from the valid range $[2, 32]$ [S1].
*   **Overflow:** $v_{max}$ is calibrated specifically to handle overflow occurrences [S1].
*   **Precision:** The Straight-Through Estimator (STE) is used for the rounding operation $\lfloor \cdot \rceil$, assuming $\partial \lfloor x \rceil / \partial x = 1$ [S1].

## Subtopics

### [Adversarial Content-Aware Quantization (A-CAQ)](16784-adversarial-content-aware-quantization-a-caq.md)
Adversarial Content-Aware Quantization (A-CAQ) is a multi-task learning method that jointly optimizes radiance field parameters and their integer bitwidths to achieve content-aware compression. It solves the accuracy degradation of Post-Training Quantization (PTQ) at low bitwidths by formulating bitwidth selection as an adversarial game against the rendering loss. Unlike direct bitwidth search in QAT space, A-CAQ uses a differentiable adversarial objective to mitigate the performance gap between PTQ and QAT.

### Component-Specific Quantization Schemes
Three distinct quantization configurations are applied based on statistical properties [S1]:
1.  **Neural Weights:** Symmetric signed quantization with range $[-2^{B-1}, 2^{B-1}-1]$ [S1]. Parameters $v_{max}$ and $r_v$ are N/A [S1]. Bitwidth $b$ is trainable [S1].
2.  **ReLU and Exponential Activations:** Asymmetric unsigned quantization with range $[0, 2^B - 1]$ [S1]. Parameters $v_{max}$ and $r_v$ are trainable [S1]. Bitwidth $b$ is trainable [S1].
3.  **Positional Encoding (PE) and Others:** Asymmetric unsigned quantization with range $[0, 2^B - 1]$ [S1]. Parameters $v_{max}$ and $r_v$ are trainable [S1]. Bitwidth $b$ is trainable [S1]. This configuration uses $v_{max}$ to handle components lacking significant statistical features [S1].

## Operating characteristics
*   **Complexity:** Time complexity is dominated by the training loop with fake quantization simulation [S1]. Memory complexity is reduced by storing lower-bitwidth parameters and activations [S1].
*   **Performance:** LBQ aligns model complexity with scene intricacy [S1]. Complex scenes use higher bitwidths, simple scenes use lower bitwidths [S1].
*   **Strengths:** Eliminates trial-and-error bitwidth selection [S1]. Differentiable bitwidth allows end-to-end optimization [S1]. Content-aware compression [S1].
*   **Limitations:** Binary quantization (1-bit) is ineffective [S1]. Requires careful initialization [S1].
*   **When to Use:** When optimizing neural radiance fields for deployment on hardware with limited memory/bandwidth, and when scene complexity varies [S1].
*   **When Not to Use:** When fixed-bitwidth inference is required without retraining, or when 1-bit quantization is desired (not supported) [S1].

## Background
LBQ was introduced in "Content-Aware Radiance Fields: Aligning Model Complexity with Scene Intricacy Through Learned Bitwidth Quantization" [S1]. It builds on QAT methods like LSQ and LSQ+, which are limited to fixed bitwidths [S1]. LBQ extends this by making bitwidth a continuous variable [S1].

## Sources
[S1] Content-Aware Radiance Fields: Aligning Model Complexity with Scene Intricacy Through Learned Bitwidth Quantization, https://doi.org/10.1007/978-3-031-72775-7_14, fetched 2026-10-26
