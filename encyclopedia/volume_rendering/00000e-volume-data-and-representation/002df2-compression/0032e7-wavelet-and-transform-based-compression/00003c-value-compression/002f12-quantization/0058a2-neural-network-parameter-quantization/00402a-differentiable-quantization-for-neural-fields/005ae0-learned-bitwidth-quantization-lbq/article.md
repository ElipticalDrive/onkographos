# Learned Bitwidth Quantization (LBQ)


## Overview
Learned Bitwidth Quantization (LBQ) is a differentiable quantization framework for neural radiance fields that treats integer bitwidth as a continuous, learnable variable ("soft bitwidth") rather than a fixed hyperparameter [S1]. It solves the accuracy-efficiency trade-off by automatically aligning model complexity with scene intricacy, eliminating the need for trial-and-error bitwidth selection required by traditional Quantization-Aware Training (QAT) [S1]. Unlike fixed-bitwidth QAT methods like LSQ, LBQ allows bitwidth to adapt per-component during training via gradient descent [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A radiance field model (e.g., Instant-NGP) with parameters $\Theta$ (multi-resolution feature table) and $\Phi$ (MLP weights), and training data (rays $\mathbf{r}$ with origin $o$ and direction $d$).
*   **Output:** A quantized model where each component has an optimized integer bitwidth $B$, range scale $r_v$, and offset $v_{max}$, producing rendered images $\mathbf{C}(\mathbf{r})$ with minimal loss compared to full precision.

**Data Structures:**
The radiance field maps 5D input $(\mathbf{x}, \mathbf{d})$ to opacity $\sigma$ and color $\mathbf{c}$ [S1]:
$$ F: (\mathbf{x}, \mathbf{d}) \xrightarrow{} (\sigma , \mathbf{c}) $$
where $\mathbf{x}$ is 3D location and $\mathbf{d}$ is 2D view direction [S1].
1.  **Encoding:** Location $\mathbf{x}$ is encoded as $\mathbf{y}_x = \text{enc}(\mathbf{x}, \Theta)$. View direction $\mathbf{d}$ is encoded as $\mathbf{y}_d$ using spherical harmonics [S1].
2.  **MLP:** The encoded vector $\mathbf{y} = (\mathbf{y}_x, \mathbf{y}_d)$ is fed to an MLP to produce $(\sigma , \mathbf{c}) = m(\mathbf{y}, \mathbf{\Phi})$ [S1].
3.  **Rendering:** Expected color along ray $\mathbf{r}$ is:
    $$ \mathbf{C}(\mathbf{r}) = \int ^{t_\text {f}}_{t_\text {n}}T(t) \sigma ( \mathbf{r}(t) ) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) dt $$
    where $T(t)= \exp (-\int ^t_{t_{\text {n}}} \sigma ( \mathbf{r}(s) ) ds )$ and $\mathbf{r}(t) = o + t\mathbf{d}$ [S1].

**Quantization Parameters:**
Each component (weights, activations) is associated with:
*   $b$: Soft bitwidth (float).
*   $B$: Integer bitwidth, $B = \lfloor b \rceil$ (round to nearest integer) [S1].
*   $r_v$: Range scale for input data $v$ (trainable) [S1].
*   $v_{max}$: Offset for quantization range (trainable) [S1].
*   $r_q$: Quantization range scale, $r_q = 2^B - 1$ [S1].
*   $Z$: Zero-point offset, $Z = \lfloor q_{\text {max}} - \frac{v_{\text {max}}}{r_v} r_q \rceil$ [S1]. Note: $q_{\text{max}}$ depends on the quantization scheme (signed/unsigned) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def clamp(x, a, c):
    if x < a:
        return a
    elif x > c:
        return c
    else:
        return x

def round_to_nearest(x):
    # Straight-Through Estimator: gradient is 1, value is rounded
    return round(x)

def fake_quantize(v, b, rv, vmax, q_min, q_max):
    """
    Simulates quantization and de-quantization for differentiable training.
    v: Input data tensor.
    b: Soft bitwidth (float).
    rv: Range scale (float).
    vmax: Offset (float).
    q_min, q_max: Quantization bounds (integers).
    """
    # Calculate integer bitwidth
    B = round_to_nearest(b)
    
    # Calculate quantization range scale
    rq = 2**B - 1
    
    # Calculate step size s = rv / rq
    s = rv / rq
    
    # Calculate zero-point offset Z
    # Z = floor(q_max - vmax / s)
    # Note: In the source, Z is derived from vmax, rv, rq.
    # Z = floor(q_max - (vmax * rq) / rv)
    Z = math.floor(q_max - (vmax * rq) / rv)
    
    # Quantize: round(v / s) + Z, then clamp
    # Using STE for gradient flow through round
    V_hat = round_to_nearest(v / s) + Z
    V_clamped = clamp(V_hat, q_min, q_max)
    
    # De-quantize
    v_hat = s * (V_clamped - Z)
    
    return v_hat

def compute_gradients(v, b, rv, vmax, q_min, q_max):
    """
    Computes gradients for trainable parameters b, rv, vmax.
    Uses analytical derivatives from Table 1 in [S1].
    """
    B = round_to_nearest(b)
    rq = 2**B - 1
    s = rv / rq
    
    # Determine region for v to select correct derivative for rv
    # vmin is implicitly handled by the clamp bounds and Z logic,
    # but the source defines derivatives based on v relative to vmax and vmin.
    # For simplicity in implementation, we check bounds.
    # Note: The source defines vmin/vmax in the context of the range [vmin, vmax].
    # In the specific LBQ formulation, vmax is the upper bound parameter.
    # The lower bound is determined by q_min and Z.
    
    # Derivative w.r.t rv
    # Case 1: vmin <= v <= vmax
    # Case 2: v > vmax
    # Case 3: v < vmin
    # We need to determine vmin. 
    # From Eq 4 and 7, the range is roughly [vmax - rv, vmax] for unsigned?
    # Actually, the source Table 1 gives derivatives based on v's position relative to 
    # the active range boundaries. 
    # Let's assume standard bounds for the specific quantization scheme.
    
    # For Neural Weights (Symmetric Signed): q_min = -2^(B-1), q_max = 2^(B-1)-1
    # For Activations (Asymmetric Unsigned): q_min = 0, q_max = 2^B - 1
    
    # Calculate vmin based on q_min, Z, s
    # v_min = s * (q_min - Z)
    v_min = s * (q_min - Z)
    v_max_bound = s * (q_max - Z) # This is effectively vmax in the derivative context?
    # The source uses vmax as the parameter. Let's stick to the source's variable names.
    # Source Table 1 uses vmax and vmin as the boundaries of the data distribution 
    # or the quantization range limits.
    
    # Derivative w.r.t b
    # d(v_hat)/d(b) = (v - s * round(v/s)) * 2^B * ln(2) / rq
    # This applies when v is within the non-clamped range.
    
    # Derivative w.r.t vmax
    # d(v_hat)/d(vmax) = 0 if in range, else (vmax - rv + s*Z)*2^B*ln(2)/rq or 1
    
    # Implementation detail: The gradients are used in the optimizer step.
    pass

def train_step(model, rays, targets):
    """
    One step of LBQ training.
    """
    # 1. Forward pass with fake quantizers
    # Insert fake quantizers into encoding and MLPs
    # Parameters: b, rv, vmax for each component
    
    # 2. Render image
    C_r = render(model, rays)
    
    # 3. Compute Loss
    loss = mse_loss(C_r, targets)
    
    # 4. Backward pass
    # Gradients flow through fake_quantize using STE and analytical derivatives
    # Update b, rv, vmax, and model weights
    optimizer.step()
```

**Steps:**
1.  **Insert Fake Quantizers:** Place fake quantizers in encoding and MLP components [S1].
2.  **Initialize Parameters:** Initialize all components to 8-bit, except exponential activation to 32-bit [S1]. Initialize $r_v$ and $v_{max}$ using Post-Training Quantization (PTQ) [S1].
3.  **Simulate Quantization:** Perform quantize and de-quantize on data $v$ using Eq. (4) [S1].
4.  **Compute Loss:** Calculate rendering loss $\mathcal{L}^{\text{NeRF}}$ [S1].
5.  **Update Parameters:** Use gradient descent to update $b$, $r_v$, $v_{max}$, and model weights. The gradients for $b$ and $r_v$ depend on the quantization error term $(v - s \cdot \lfloor v/s \rceil)$ [S1]. $v_{max}$ is calibrated upon overflow [S1].

**Parameters:**
*   $b$: Soft bitwidth. Initialized to 8.0 for most components, 32.0 for exponential activation [S1].
*   $r_v$: Range scale. Initialized via PTQ [S1].
*   $v_{max}$: Offset. Initialized via PTQ [S1].
*   Bitwidth Range: Constrained to $[2, 32]$. Binary (1-bit) is excluded as it yields nonsensical results [S1].

**Edge Cases and Pitfalls:**
*   **Binary Quantization:** 1-bit quantization produces nonsensical rendering and is excluded from the valid range $[2, 32]$ [S1].
*   **Overflow:** $v_{max}$ is calibrated specifically to handle overflow occurrences [S1].
*   **Precision:** The Straight-Through Estimator (STE) is used for the rounding operation $\lfloor \cdot \rceil$, assuming $\partial \lfloor x \rceil / \partial x = 1$ [S1].

## Subtopics

### [Adversarial Content-Aware Quantization (A-CAQ)](16784-adversarial-content-aware-quantization-a-caq.md)
Adversarial Content-Aware Quantization (A-CAQ) is a multi-task learning method that jointly optimizes radiance field parameters and their integer bitwidths to achieve content-aware compression. It solves the accuracy degradation of Post-Training Quantization (PTQ) at low bitwidths by formulating bitwidth selection as an adversarial game against the rendering loss. Unlike direct bitwidth search in QAT space, A-CAQ uses a differentiable adversarial objective to mitigate the performance gap between PTQ and QAT.

### Component-Specific Quantization Schemes
Three distinct quantization configurations are applied based on statistical properties [S1]:
1.  **Neural Weights:** Symmetric signed quantization with range $[-2^{B-1}, 2^{B-1}-1]$. Parameters $v_{max}$ and $r_v$ are N/A. Bitwidth $b$ is trainable [S1].
2.  **ReLU and Exponential Activations:** Asymmetric unsigned quantization with range $[0, 2^B - 1]$. Parameters $v_{max}$ and $r_v$ are trainable. Bitwidth $b$ is trainable [S1].
3.  **Positional Encoding (PE) and Others:** Asymmetric unsigned quantization with range $[0, 2^B - 1]$. Parameters $v_{max}$ and $r_v$ are trainable. Bitwidth $b$ is trainable. This uses $v_{max}$ to handle components lacking significant statistical features [S1].

## Operating characteristics
*   **Complexity:** Time complexity is dominated by the training loop with fake quantization simulation. Memory complexity is reduced by storing lower-bitwidth parameters and activations [S1].
*   **Performance:** LBQ aligns model complexity with scene intricacy. Complex scenes use higher bitwidths, simple scenes use lower bitwidths [S1].
*   **Strengths:** Eliminates trial-and-error bitwidth selection. Differentiable bitwidth allows end-to-end optimization. Content-aware compression [S1].
*   **Limitations:** Binary quantization (1-bit) is ineffective. Requires careful initialization (PTQ) [S1].
*   **When to Use:** When optimizing neural radiance fields for deployment on hardware with limited memory/bandwidth, and when scene complexity varies [S1].
*   **When Not to Use:** When fixed-bitwidth inference is required without retraining, or when 1-bit quantization is desired (not supported) [S1].

## Background
LBQ was introduced in "Content-Aware Radiance Fields: Aligning Model Complexity with Scene Intricacy Through Learned Bitwidth Quantization" [S1]. It builds on QAT methods like LSQ and LSQ+, which are limited to fixed bitwidths [S1]. LBQ extends this by making bitwidth a continuous variable [S1].

## Sources
[S1] Content-Aware Radiance Fields: Aligning Model Complexity with Scene Intricacy Through Learned Bitwidth Quantization, https://doi.org/10.1007/978-3-031-72775-7_14, fetched 2026-10-26
