# Quantization with Noise Injection

Also called: Noise-based Quantization, Noise-injected attribute quantization, Quantization Noise Simulation, Quantization-aware training with noise injection, Simulated Quantization, Simulated Quantization for Gradient Propagation

## Overview
Quantization with Noise Injection compresses continuous attributes into discrete values for entropy coding by simulating quantization error during training via uniform noise injection, while applying explicit rounding during inference [S1, unofficial][S2]. It differs from standard quantization by preserving gradient flow through the otherwise non-differentiable rounding operation, enabling end-to-end training with quantization constraints [S1, unofficial][S2][S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous attribute components $f_i$ (e.g., $f_a$, $l_s$, $l_r$, $\Delta x_g$) and a predefined quantization step $Q$ [S1, unofficial].
*   **Output:** Quantized attribute values $\hat{f}_i$ for entropy coding [S1, unofficial].

**Data Structures:**
Attributes are stored as continuous floating-point values during training and converted to discrete representations for storage [S1, unofficial]. The specific attributes quantized in the primary implementation are $f_a$ (appearance), $l_s$ (scale), $l_r$ (rotation), and $\Delta x_g$ (position) [S1, unofficial].

**Steps:**
1.  **Compute Quantization Step:** Calculate the adaptive quantization step size $q_i$ for each attribute component $i$ using a refinement factor $r_i$ output from a context model [S1, unofficial].
2.  **Training Phase (Noise Injection):** Sample uniform noise $\mathcal{V} \sim U[-0.5, 0.5]$ [S1, unofficial]. Inject noise into the attribute value to simulate quantization error: $\hat{f}_i = f_i + \mathcal{V} \times q_i$ [S1, unofficial]. Use $\hat{f}_i$ for the forward pass and backpropagation, substituting the original $f_i$ [S1, unofficial]. Alternatively, in some implementations, uniform noise $\epsilon \sim U[-0.5, 0.5]$ is added directly to the encoder output $E(x; \theta_e)$ to simulate quantization error $\hat{z} = E(x; \theta_e) + \epsilon$ [S2].
3.  **Inference Phase (Explicit Quantization):** Compute the quantized value using rounding: $\hat{f}_i = \text{round}(f_i / q_i) \times q_i$ [S1, unofficial]. Pass the discrete $\hat{f}_i$ to the entropy coder [S1, unofficial][S2].

**Formulas:**
The quantization step size $q_i$ is defined as [S1, unofficial]:
$$ q_i = Q \times (1 + \tanh(r_i)) $$
where:
*   $Q$ is the predefined base quantization step size [S1, unofficial].
*   $r_i$ is the refinement factor from the context model [S1, unofficial].

The training quantization is [S1, unofficial]:
$$ \hat{f}_i = f_i + \mathcal{V}(-\frac{1}{2}, \frac{1}{2}) \times q_i $$
where $\mathcal{V}$ represents uniform sampling [S1, unofficial].

The testing quantization is [S1, unofficial]:
$$ \hat{f}_i = \text{round}\left(\frac{f_i}{q_i}\right) \times q_i $$

**Parameters:**
Typical values for the predefined quantization step $Q$ vary by attribute component due to different numerical scales [S1, unofficial]:
*   $Q = 1$ for $f_a$ (appearance) [S1, unofficial].
*   $Q = 0.001$ for $l_s$ (scale) [S1, unofficial].
*   $Q = 0.001$ for $l_r$ (rotation) [S1, unofficial].
*   $Q = 0.2$ for $\Delta x_g$ (position) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The refinement factor $r_i$ allows adaptive adjustment of $q_i$, which is critical because attributes exhibit different statistical distributions and scales [S1, unofficial].
*   **Distribution Assumption:** The method assumes attributes exhibit Gaussian-like distributions, allowing the context model to output individual mean $\mu_i$ and variance $\sigma_i$ for probability modeling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_quantization_step(Q, r_i):
    return Q * (1 + math.tanh(r_i))

def quantize_training(f_i, q_i):
    noise = random.uniform(-0.5, 0.5)
    return f_i + noise * q_i

def quantize_inference(f_i, q_i):
    return round(f_i / q_i) * q_i
```

## Subtopics

### Post-Training Compression
A technique that reduces storage requirements by quantizing vertex positions, features, and MLP weights after training, followed by entropy coding [S5]. It quantizes vertex positions to UInt16, vertex features to uniform Int8 (with a per-channel scale and offset), and MLP weights to FP16 [S5]. An entropy coder (Zstandard, level 19) is applied to the serialized payload [S5]. This achieves an additional compression factor of approximately 3x beyond the training result with negligible quality loss [S5].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Significantly reduces memory footprint by storing discrete quantized values instead of full-precision floats, with further gains from entropy coding [S1, unofficial][S5].
*   **Strengths:** Enables end-to-end training with quantization constraints, allowing gradient propagation through the non-differentiable quantization operation [S1, unofficial][S2][S3].
*   **Limitations:** Requires careful tuning of $Q$ and the context model for refinement factors [S1, unofficial].

## Background
The technique originates from the need to compress neural representations where continuous latent codes must be discretized for entropy coding [S1, unofficial][S2]. Ballé et al. [S2] proposed simulating quantization noise during training to enable backpropagation through the quantization operation. This approach has been adapted for compressing anchor attributes in volume visualization [S1, unofficial] and dynamic neural radiance fields [S3][S4].

## Sources
[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
[S2] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
[S3] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S4] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
[S5] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
