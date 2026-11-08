# Quantization with Noise Injection

Also called: Noise-injected attribute quantization, Quantization-aware training with noise injection, Quantization Noise Simulation

## Overview
Quantization with Noise Injection is a technique to compress continuous anchor attributes into discrete values suitable for entropy coding by simulating quantization error during training [S1, unofficial][S2]. It differs from explicit post-training quantization by allowing gradients to flow through the quantization operation via noise addition, enabling end-to-end optimization of the compression rate-distortion trade-off [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous attribute component $f_i$ (e.g., color, normal, position delta) for an anchor $A_i$.
*   **Output:** Quantized attribute $\hat{f}_i$ used for training or final encoding.

**Data Structures:**
*   Attributes are treated as real-valued vectors.
*   Quantization step $q_i$ is a scalar derived from a predefined base $Q$ and a learned refinement factor $r_i$.

**Parameters:**
*   $Q$: Predefined quantization step size. Values vary by component:
    *   $Q = 1$ for $f_a$ (anchor attributes, e.g., color) [S1, unofficial].
    *   $Q = 0.001$ for $l_s$ (scaling factors) [S1, unofficial].
    *   $Q = 0.001$ for $l_r$ (rotation factors) [S1, unofficial].
    *   $Q = 0.2$ for $\Delta x_g$ (position deltas) [S1, unofficial].
*   $\epsilon$: Uniform noise sampled from $U[-0.5, 0.5]$ [S2].
*   $r_i$: Refinement factor output from a context model [S1, unofficial].

**Steps:**

1.  **Compute Quantization Step:** Calculate the adaptive step size $q_i$ using the predefined base $Q$ and the context model's refinement factor $r_i$:
    $$ q_i = Q \times (1 + \tanh(r_i)) $$
    [S1, unofficial]

2.  **Training Phase (Noise Injection):** To enable backpropagation, substitute the original attribute $f_i$ with a noisy version $\hat{f}_i$ during the forward pass. Sample noise $\mathcal{V} \sim U(-0.5, 0.5)$ and compute:
    $$ \hat{f}_i = f_i + \mathcal{V} \times q_i $$
    [S1, unofficial]
    This $\hat{f}_i$ is used for subsequent loss computation and gradient updates. The noise simulates the rounding error of explicit quantization [S2].

3.  **Inference/Testing Phase (Explicit Quantization):** During encoding, explicitly quantize the attribute by rounding to the nearest multiple of $q_i$:
    $$ \hat{f}_i = \text{round}\left(\frac{f_i}{q_i}\right) \times q_i $$
    [S1, unofficial]
    The resulting discrete values are then passed to an entropy coder [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_quantization_step(Q_base, refinement_factor):
    """
    Computes the adaptive quantization step size.
    Q_base: Predefined step size for the attribute component.
    refinement_factor: Output from the context model (r_i).
    """
    # tanh is the hyperbolic tangent function
    return Q_base * (1.0 + math.tanh(refinement_factor))

def quantize_with_noise_injection(f_i, q_i, is_training):
    """
    Performs quantization with noise injection during training,
    and explicit rounding during inference.
    
    f_i: Continuous attribute component value.
    q_i: Quantization step size.
    is_training: Boolean flag indicating training or inference mode.
    """
    if is_training:
        # Sample uniform noise from U(-0.5, 0.5)
        # Note: random.uniform(a, b) returns a float N such that a <= N <= b
        noise = random.uniform(-0.5, 0.5)
        # Add noise scaled by step size
        f_hat = f_i + noise * q_i
        return f_hat
    else:
        # Explicit rounding for inference/encoding
        # round() rounds to the nearest integer
        f_hat = round(f_i / q_i) * q_i
        return f_hat

# Example usage for training
Q_fa = 1.0
Q_ls = 0.001
Q_lr = 0.001
Q_dxg = 0.2

# Assume context model provides refinement factors
r_fa = 0.1
r_ls = 0.0
r_lr = 0.0
r_dxg = 0.5

q_fa = compute_quantization_step(Q_fa, r_fa)
q_ls = compute_quantization_step(Q_ls, r_ls)
q_lr = compute_quantization_step(Q_lr, r_lr)
q_dxg = compute_quantization_step(Q_dxg, r_dxg)

# During training, use noisy quantization
f_a_original = 0.5
f_a_train = quantize_with_noise_injection(f_a_original, q_fa, is_training=True)

# During inference, use explicit quantization
f_a_infer = quantize_with_noise_injection(f_a_original, q_fa, is_training=False)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The refinement factor $r_i$ must be learned effectively; poor convergence may lead to suboptimal $q_i$ values [S1, unofficial].
*   **Distribution Assumption:** The method assumes attributes exhibit statistical Gaussian-like distributions to model probabilities $p(\hat{f}_i)$ using Gaussian distributions $\mathcal{N}(\mu_i, \sigma_i)$ [S1, unofficial]. Non-Gaussian attributes may require different probability models.
*   **Noise Range:** The noise is strictly bounded to $[-0.5, 0.5]$ scaled by $q_i$, ensuring the error does not exceed half a quantization step [S2].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Training requires sampling noise and computing $\tanh$, which is negligible compared to encoder/decoder forward passes. Inference requires rounding and entropy coding.
*   **Memory Complexity:** No additional memory overhead during training compared to standard autoencoders. The entropy coder requires storage for the discrete codes.
*   **Strengths:** Enables end-to-end optimization of quantization parameters; avoids the gradient discontinuity of straight-through estimator or explicit rounding during training [S1, unofficial][S2].
*   **Limitations:** Relies on the context model to predict appropriate refinement factors; performance depends on the accuracy of the Gaussian probability model [S1, unofficial].
*   **When to use:** When compressing neural representations (e.g., Gaussian splats, implicit volumes) where attributes have varying scales and need efficient entropy coding [S1, unofficial][S2].

## Background
The technique was proposed by Ballé et al. [2] (cited in [S2]) to simulate quantization noise during training rather than explicitly discretizing the code, allowing gradients to flow. It has been applied in recent work on compressive neural Gaussian splats (ECoNGS) [S1, unofficial] and deep implicit volume compression [S2].

## Sources
[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
[S2] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
