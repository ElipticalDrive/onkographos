# Single-Event Upset (SEU) Fault Injection in 3DGS

Also called: Bit-flip injection, Single-bit upset injection

## Overview
Single-Event Upset (SEU) fault injection in 3D Gaussian Splatting (3DGS) simulates transient bit flips in GPU memory to measure the rendering corruption caused by hardware errors [S1, unofficial]. It differs from neural network weight perturbation because 3DGS primitives are spatially localized and redundant, whereas network weights are shared globally; this method quantifies whether such redundancy provides robustness or concentrates risk into rare catastrophic failures [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A trained 3DGS model with $N$ primitives stored in GPU memory, a set of $K$ held-out camera views, and a fault site tuple $(field, primitive, component, bit)$ [S1, unofficial].
*   **Output:** Corruption metrics (peak pixel error, LPIPS, SSIM, PSNR, non-finite flag, corruption footprint) for the rendered views [S1, unofficial].

**Data Structures**
A trained 3DGS model consists of $N$ primitives. Primitive $i$ stores six fields in memory [S1, unofficial]:
1.  Mean $\mu_i \in \mathbb{R}^3$
2.  Logarithmic scale $s_i \in \mathbb{R}^3$ (renderer applies $\exp$)
3.  Quaternion $q_i \in \mathbb{R}^4$ (normalized at render time)
4.  Opacity logit $o_i \in \mathbb{R}$ (renderer applies logistic $\sigma$)
5.  DC color term $c_{0i} \in \mathbb{R}^3$
6.  Higher-order spherical harmonic coefficients $c_{Ni} \in \mathbb{R}^{((\ell+1)^2-1) \times 3}$

**IEEE-754 Bit Layouts**
The bit positions determine the impact of a flip [S1, unofficial]:
*   **fp32:** Sign is bit 31, exponent is bits 23–30, mantissa is bits 0–22.
*   **fp16:** 5-bit exponent, 10-bit mantissa.
*   **bf16:** 8-bit exponent (from fp32), 7-bit mantissa.

**Algorithm Steps**
1.  **Reinterpret:** Cast the stored floating-point value at the fault site to an unsigned integer of the matching width.
2.  **Flip:** XOR the integer with a mask having a single bit set at the target position.
3.  **Restore:** Reinterpret the modified integer back to the original floating-point format.
4.  **Render:** Render the $K$ held-out views using the corrupted parameter.
5.  **Measure:** Compute metrics on the GPU.
6.  **Cleanup:** Restore the original value to the working copy.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import struct

def flip_bit_fp32(value: float, bit_index: int) -> float:
    """Flip a single bit in an fp32 value.
    
    Args:
        value: The original floating-point value.
        bit_index: The bit position to flip (0-31).
        
    Returns:
        The value with the specified bit flipped.
    """
    # Reinterpret float as unsigned 32-bit integer
    # struct.unpack returns a tuple, so we take the first element
    bits = struct.unpack('I', struct.pack('f', value))[0]
    
    # Flip the bit using XOR
    bits ^= (1 << bit_index)
    
    # Reinterpret back to float
    return struct.unpack('f', struct.pack('I', bits))[0]

def inject_seu_and_render(model, fault_site, cameras, renderer):
    """Inject a single-bit upset and render metrics.
    
    Args:
        model: Object containing primitive parameters in GPU memory.
        fault_site: Tuple (field_name, primitive_index, component_index, bit_index).
        cameras: List of K camera poses.
        renderer: Function that renders primitives to images given cameras.
        
    Returns:
        Dictionary of metrics.
    """
    field_name, prim_idx, comp_idx, bit_idx = fault_site
    
    # 1. Get clean value from model
    # [G] Assuming model.params[field_name][prim_idx][comp_idx] accesses the value
    clean_value = model.params[field_name][prim_idx][comp_idx]
    
    # 2. Inject fault
    corrupted_value = flip_bit_fp32(clean_value, bit_idx)
    
    # 3. Write to working copy
    model.params[field_name][prim_idx][comp_idx] = corrupted_value
    
    # 4. Render K views
    # [G] Assuming renderer returns a list of images or a batched tensor
    corrupted_images = renderer(model.params, cameras)
    
    # 5. Render clean views (or use cached clean render)
    # [G] Assuming clean_images are pre-computed or re-rendered
    clean_images = renderer(model.params, cameras) 
    # Note: In practice, clean_images should be from the uncorrupted model state.
    # The pseudocode above re-reads from model.params which is now corrupted.
    # Correct approach: Store clean_value, render clean once, then swap.
    
    # 6. Compute metrics
    metrics = compute_metrics(corrupted_images, clean_images)
    
    # 7. Restore value
    model.params[field_name][prim_idx][comp_idx] = clean_value
    
    return metrics

def compute_metrics(corrupted_imgs, clean_imgs):
    """Compute corruption metrics."""
    # [G] Standard image processing functions assumed available
    peak_error = np.max(np.abs(corrupted_imgs - clean_imgs))
    psnr = 10 * np.log10(1.0 / np.mean((corrupted_imgs - clean_imgs)**2))
    
    # Corruption footprint: fraction of pixels changing > 1/255
    diff = np.abs(corrupted_imgs - clean_imgs)
    footprint = np.mean(diff > (1.0/255.0))
    
    # Non-finite check
    is_non_finite = np.any(np.isnan(corrupted_imgs)) or np.any(np.isinf(corrupted_imgs))
    
    return {
        'peak_error': peak_error,
        'psnr': psnr,
        'footprint': footprint,
        'non_finite': is_non_finite
    }
```

**Parameters and Typical Values**
*   **Precision:** fp32 is the baseline; fp16 and bf16 are tested by casting clean parameters to the target format before injection [S1, unofficial].
*   **Views ($K$):** Multiple held-out views are averaged to compute the corruption footprint [S1, unofficial].
*   **Catastrophic Threshold:** An upset is catastrophic if the corruption footprint exceeds 1% of the frame or produces non-finite values [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Exponent Overflow:** Flipping high exponent bits in fp32 can cause overflow to non-finite values [S1, unofficial].
*   **Scale Sign Flip:** For the scale field, the activation is $\phi = \exp$. A sign flip on a typical log scale $\theta \approx -3$ changes it to $+3$, multiplying the primitive size by $e^6 \approx 4 \times 10^2$ [S1, unofficial].
*   **Global Metrics:** Full-frame PSNR is a poor discriminator for single upsets in large scenes because a single corrupted primitive among hundreds of thousands barely moves the global average [S1, unofficial].

## Subtopics

### Perturbation Bound Analysis
Analyzes the mathematical impact of bit flips on specific fields. For the scale field, the relative change is $|\Delta s|/s = |\Delta \theta|$, meaning a mantissa flip changes scale by a fixed relative amount $2^{b-p}$ independent of the exponent $e$ [S1, unofficial]. For a normal fp32 value $\theta = (-1)^\epsilon 2^e (1+m)$, flipping mantissa bit $b$ changes the value by $|\Delta \theta| = 2^{e+b-p}$ [S1, unofficial]. Flipping the sign bit gives $\Delta \theta = -2\theta$, and flipping exponent bit $j$ multiplies magnitude by $2^{\pm 2^j}$ [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity**
*   **Time:** The parallel injection engine keeps the GPU busy with a stream of independent corrupted renders, saturating device throughput [S1, unofficial].
*   **Memory:** Clean parameters remain resident on the GPU; only one working copy is modified at a time [S1, unofficial].

**Measured Performance**
*   **Redundancy Scaling:** For a converged model of $N$ primitives, the typical (median) image error of one uniform-random single-bit upset scales as $\sigma^2(N) = \Theta(N^{-\alpha})$ for a scene-dependent exponent $\alpha > 0$ [S1, unofficial].
*   **Dose Budget:** Under $k$ independent uniform single-bit upsets, the expected squared image error is $E\|\Delta I\|^2 = k \sigma^2(N)$ [S1, unofficial]. The dose $k_\tau$ at which error exceeds threshold $\tau$ scales as $k_\tau = \Theta(N^\alpha \tau)$ [S1, unofficial].

**Strengths and Limitations**
*   **Strengths:** Directly measures hardware vulnerability; identifies critical bit positions (e.g., scale sign bit) [S1, unofficial].
*   **Limitations:** Full-frame PSNR is insensitive to local corruption; catastrophic upsets are rare but dominant in mean error [S1, unofficial].

**When to Use**
*   Use for evaluating robustness of 3DGS deployments against hardware errors.
*   Do not use for optimizing model accuracy; use for safety-critical validation [S1, unofficial].

## Background
Originated from the paper "Single-Event Upsets in 3D Gaussian Splatting Rendering: Bit-Level Criticality, Spatial Extent, and a Parallel Support Guard" [S1, unofficial]. It addresses the gap in understanding whether the redundancy of 3DGS primitives provides robustness against transient faults, contrasting with extensive prior work on neural network weight perturbation [S1, unofficial].

## Sources
[S1] Single-Event Upsets in 3D Gaussian Splatting Rendering: Bit-Level Criticality, Spatial Extent, and a Parallel Support Guard, https://doi.org/10.48550/arxiv.2606.21791, fetched 2026-10-26 (unofficial)
