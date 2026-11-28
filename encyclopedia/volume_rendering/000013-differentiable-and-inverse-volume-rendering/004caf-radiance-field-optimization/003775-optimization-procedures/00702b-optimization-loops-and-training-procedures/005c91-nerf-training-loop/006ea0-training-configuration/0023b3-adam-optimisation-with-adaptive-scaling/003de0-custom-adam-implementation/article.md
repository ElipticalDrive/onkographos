# Custom Adam Implementation

Also called: Fused Custom Adam, Optimized PyTorch Adam, Fused Adam, Fused Adam optimizer

## Overview
A custom CUDA implementation of the Adam optimizer that precisely matches PyTorch's behavior while optimizing away unnecessary overhead [S1, unofficial]. It fuses the Adam update step into the training kernel to reduce kernel launch overhead and improve performance [S1, unofficial]. It sets itself apart from standard PyTorch Adam and NVIDIA Apex Fused Adam by further reducing instruction count through fused-multiply-adds and fast math operations, achieving the highest speedup among fused variants [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: Parameter tensors, gradient tensors, state tensors (`exp_avg`, `exp_avg_sq`), hyperparameters ($\beta_1, \beta_2, \epsilon, \text{lr}$), and step count [G].
- Outputs: Updated parameter tensors [G].

**Data Structures:**
- State tensors are stored in device memory [G].
- Locality-preserving densification applies z-ordering to Gaussian parameters to ensure spatially close Gaussians are stored contiguously, preventing uncoalesced memory accesses during densification [S1, unofficial].

**Steps:**
1. **Kernel Fusion:** The Adam update logic is fused directly into the CUDA kernel of the training pass, eliminating separate kernel launches for the optimizer step [S1, unofficial].
2. **Fast Math:** CUDA fast math operations are used to accelerate division and square root calculations [S1, unofficial].
3. **Fused-Multiply-Add (FMA):** Multiplication and addition operations are combined into single instructions to reduce overall instruction count and improve throughput [S1, unofficial].
4. **State Update:** Exponential moving averages and bias-corrected estimates are computed using the fused logic, precisely matching PyTorch's Adam behavior [S1, unofficial].
5. **Parameter Update:** The standard Adam parameter update step is applied [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fused_adam_step(params, grads, exp_avg, exp_avg_sq, beta1, beta2, lr, eps, step, num_elements):
    # Conceptual representation of the fused CUDA kernel logic
    # Each thread handles one or more elements
    # [G] State tensors must be initialized to zero before training begins.
    for i in range(num_elements):
        g = grads[i]
        
        # Update biased first moment estimate
        exp_avg[i] = beta1 * exp_avg[i] + (1.0 - beta1) * g
        
        # Update biased second raw moment estimate
        exp_avg_sq[i] = beta2 * exp_avg_sq[i] + (1.0 - beta2) * (g * g)
        
        # Compute bias-corrected first moment estimate
        bias_correction1 = 1.0 - beta1 ** step
        m_hat = exp_avg[i] / bias_correction1
        
        # Compute bias-corrected second raw moment estimate
        bias_correction2 = 1.0 - beta2 ** step
        v_hat = exp_avg_sq[i] / bias_correction2
        
        # Update parameters using standard Adam formula
        params[i] = params[i] - lr * m_hat / (math.sqrt(v_hat) + eps)
```

**Parameters:**
- $\beta_1$: Typically 0.9 [G].
- $\beta_2$: Typically 0.999 [G].
- $\epsilon$: Typically $10^{-8}$ [G].
- $\text{lr}$: Learning rate, varies by scene and configuration [G].

**Edge Cases and Pitfalls:**
- **Precision:** Fast math operations are used, but the implementation is designed to precisely match PyTorch's Adam behavior [S1, unofficial].
- **Memory Locality:** Without z-ordering during densification, memory accesses become uncoalesced, throttling performance [S1, unofficial].
- **State Consistency:** Each thread handles distinct elements, so atomic operations are not required [G].

## Subtopics

### [Adam Optimizer with Torch Compatibility and Tile-Level Sparsity](13981-adam-optimizer-with-torch-compatibility-and-tile-level-spars.md)
A CUDA implementation of the Adam optimizer designed for differentiable volume rendering that maintains strict behavioral compatibility with PyTorch's Adam while optimizing for tile-level sparsity. It solves the problem of inconsistent state updates in sparse rendering pipelines where some tiles receive zero gradients, by replaying optimization steps for skipped tiles to ensure exponential moving averages decay correctly. It sets itself apart from standard Adam implementations by explicitly handling the temporal gap between optimization steps for untiled regions, ensuring that the state evolution matches what would occur if every parameter were updated at every step, even with zero gradients.

### CUDA-side Adam Optimizer for SH Coefficients
A CUDA implementation of the Adam optimizer that launches a separate thread for every spherical harmonic coefficient, rather than one thread per Gaussian [S2]. It differs from EPBRR by not fusing the forward and backward pass, allowing for SSIM loss and custom losses, and by using per-coefficient threads instead of per-Gaussian threads [S2].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is reduced compared to standard PyTorch Adam due to kernel fusion and reduced overhead [S1, unofficial].
- Memory complexity is similar to standard Adam, with VRAM usage remaining at 6.40GiB, comparable to PyTorch (6.42GiB) and Apex (6.23GiB) [S1, unofficial].

**Measured Performance:**
- On Mip-NeRF360 outdoor scenes with an RTX 4090, custom fused Adam reduces training time to 12m50s, achieving a 1.33x speedup compared to the baseline, with 6.40GiB VRAM usage [S1, unofficial].
- Custom fused Adam achieves a 1.33x speedup, compared to PyTorch fused Adam (1.22x) and Apex Fused Adam (1.30x) [S1, unofficial].
- VRAM usage remains at 6.40GiB for the custom fused Adam, comparable to PyTorch (6.42GiB) and Apex (6.23GiB) [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Highest speedup among fused Adam variants, precise behavior matching, efficient memory access patterns with z-ordering [S1, unofficial].
- **Limitations:** Requires custom CUDA implementation [G].

**When to Use:**
- Not given in the sources.

**When Not to Use:**
- Not given in the sources.

## Background
Originated in the context of optimizing Gaussian Splatting training, specifically in the Faster-GS project [S1, unofficial]. It was developed to address the performance bottlenecks of standard Adam implementations in large-scale differentiable volume rendering [S1, unofficial]. The method builds on the concept of kernel fusion introduced in PyTorch and NVIDIA Apex, but further optimizes the implementation for specific rendering workloads [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
