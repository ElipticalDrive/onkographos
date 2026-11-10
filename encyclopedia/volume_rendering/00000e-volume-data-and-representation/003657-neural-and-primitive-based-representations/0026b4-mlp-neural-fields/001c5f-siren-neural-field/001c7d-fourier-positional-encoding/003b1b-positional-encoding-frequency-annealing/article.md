# Positional Encoding Frequency Annealing

Also called: Fourier Frequency Annealing, Frequency Annealing

## Overview
Positional Encoding Frequency Annealing is a coarse-to-fine optimization strategy that controls the bandwidth of a Neural Tangent Kernel (NTK) by weighting the frequency bands of Fourier positional encoding with a time-dependent window function [S1]. It prevents MLPs from overfitting to high-frequency noise or underfitting due to insufficient bandwidth by initially constraining the network to low-frequency components and gradually exposing higher frequencies [S1][S2, unofficial]. This method sets itself apart from standard Fourier Positional Encoding by introducing a temporal schedule that modulates the effective input dimensionality during training, whereas siblings like Off-Axis Positional Encoding modify the spatial orientation of frequencies and Integrated Positional Encoding modifies the statistical properties of the encoding [S1][S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Spatial coordinates $x$, optimization iteration $t$, total annealing steps $K$, number of frequency bands $m$.
- Output: Windowed positional encoding vector $\gamma_\alpha(x)$.

**Data Structures:**
- Frequency bands are indexed $j \in \{0, \dots, m-1\}$.
- The parameter $\alpha$ is a scalar that increases linearly from $0$ to $m$ over the course of optimization.

**Steps:**
1. Calculate the annealing parameter $\alpha$ based on the current iteration $t$. The sources provide two formulations for this schedule:
   - Linear schedule: $\alpha(t) = \frac{m \cdot t}{K}$, where $K$ is a hyper-parameter controlling the total duration of annealing [S2, unofficial].
   - Fixed-step schedule: Fix $\beta=0$ (see below) for a fixed number of iterations (e.g., 1000), then linearly ease in frequencies by increasing $\beta$ to 1 over a subsequent period (e.g., 10k iterations) [S1]. Note: In the context of the window function formula, $\alpha$ is the primary driver of frequency exposure.
2. Compute the window weight $w_j(\alpha)$ for each frequency band index $j$ using a truncated Hann window function:
   $$ w_j(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - j, 0, 1))}{2} $$
   where $\text{clamp}(v, 0, 1)$ restricts the value to the range $[0, 1]$ [S1].
3. Compute the standard Fourier positional encoding components for each band $j$: $\sin(2^j x)$ and $\cos(2^j x)$ [S1].
4. Apply the window weights to the sine and cosine components:
   $$ \gamma_\alpha(x) = [w_0(\alpha)\sin(x), \dots, w_{m-1}(\alpha)\cos(2^{m-1}x)]^T $$
   This effectively masks out high-frequency bands when $\alpha < j$ and fully exposes them when $\alpha \ge j+1$ [S1].

**Parameters:**
- $m$: Number of encoding frequencies. Typical values depend on the scene complexity; not explicitly fixed in sources but implied to be sufficient for the target resolution [S1][S2, unofficial].
- $K$: Hyper-parameter for the annealing schedule duration. Set to 15,000 iterations in reported experiments [S2, unofficial].
- $\beta$: A parameter used specifically for ambient coordinates in deformation fields. Setting $\beta=0$ disables identity concatenation, allowing ambient dimensions to be collapsed during initial optimization phases [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The clamp operation ensures numerical stability in the window function calculation [S1].
- **Boundaries:** When $\alpha < 0$, all weights are zero; when $\alpha \ge m$, all weights are one. The transition is smooth due to the cosine window [S1].
- **Ambient vs. Spatial:** In hyper-space neural radiance fields, windowing is applied only to ambient coordinates (with $\beta$ schedule), while spatial coordinates $x$ use their own windowing schedule or standard encoding [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def windowed_positional_encoding(positions, m, alpha):
    """
    Computes windowed positional encoding for a given alpha.
    
    Args:
        positions: Tensor of shape [..., D] containing input coordinates.
        m: Number of frequency bands.
        alpha: Current annealing parameter in [0, m].
        
    Returns:
        Tensor of shape [..., m * 2 * D] containing windowed sin/cos components.
    """
    # Generate frequency bands: 2^0, 2^1, ..., 2^(m-1)
    # levels are integers 0 to m-1
    levels = torch.arange(m, device=positions.device)
    freq_bands = 2.0 ** levels
    
    # Compute window weights for each band j
    # w_j(alpha) = (1 - cos(pi * clamp(alpha - j, 0, 1))) / 2
    # Note: alpha is a scalar or broadcastable tensor
    diff = alpha - levels
    clamped_diff = torch.clamp(diff, min=0.0, max=1.0)
    weights = (1.0 - torch.cos(math.pi * clamped_diff)) / 2.0
    
    # Expand weights to match positions shape for broadcasting
    # weights shape: [m]
    # positions shape: [..., D]
    # We need to apply weight j to the j-th frequency band of all positions
    
    # Reshape weights for broadcasting: [1, m, 1]
    weights_expanded = weights.view(1, -1, 1)
    
    # Compute frequency components for all bands at once
    # pts shape: [..., m, D]
    pts = positions[..., None, :] * freq_bands.view(1, -1, 1)
    
    # Compute sin and cos
    pts_sin = torch.sin(pts)
    pts_cos = torch.cos(pts)
    
    # Apply window weights
    # weights_expanded broadcasts over [..., m, D]
    pts_sin_windowed = pts_sin * weights_expanded
    pts_cos_windowed = pts_cos * weights_expanded
    
    # Concatenate sin and cos along the last dimension
    # Result shape: [..., m, 2*D]
    pts_combined = torch.cat([pts_sin_windowed, pts_cos_windowed], dim=-1)
    
    # Reshape to final dimension: [..., m * 2 * D]
    final_shape = list(positions.shape[:-1]) + [m * 2 * positions.shape[-1]]
    pts_final = pts_combined.view(final_shape)
    
    return pts_final
```

## Subtopics

### Positional Encoding with Linear Annealing
A variant that applies linear annealing to frequency bands based on a progress parameter, masking lower frequencies at progress=0 and exposing all at progress=1 [S3, unofficial]. It creates frequency bands as powers of 2 ($2^{\text{levels}}$) and computes a mask using $\text{clamp}(\text{progress} \cdot \text{freqs} - \text{levels}, 0.0, 1.0)$ [S3, unofficial]. This mask is multiplied directly with the sine and cosine components before concatenation [S3, unofficial]. The parameter `progress` controls exposure, where 0 means fully masked and 1 means fully exposed [S3, unofficial]. This approach is simpler than the Hann window but achieves similar coarse-to-fine regularization [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(m \cdot D)$ per evaluation, where $m$ is the number of frequency bands and $D$ is the input dimension. The windowing adds negligible overhead compared to standard positional encoding [S1][S3, unofficial].
- Memory: $O(m \cdot D)$ to store the encoded vector. No additional memory is required for the schedule parameters [S1].

**Measured Performance:**
- In NeRF-Synthetic and LLFF datasets, this method improves PSNR and SSIM compared to baselines like standard NeRF and mip-NeRF when used in few-shot settings (e.g., GeCoNeRF) [S2, unofficial].
- It prevents getting stuck in sub-optimal local minima in deformation fields (Nerfies) by retaining the ability to represent high-frequency deformations while avoiding early overfitting [S1].

**Scaling:**
- Scales linearly with the number of frequency bands $m$ [S1].
- The annealing schedule $K$ should be scaled with the complexity of the scene; larger scenes may require larger $K$ [S2, unofficial].

**Strengths and Limitations:**
- Strengths: Regularizes optimization, prevents high-frequency artifacts and self-occlusions in early training stages, and allows coarse-to-fine learning [S1][S2, unofficial].
- Limitations: Requires careful tuning of $K$ and $m$; improper scheduling can lead to underfitting if high frequencies are not exposed sufficiently [S1][S2, unofficial].

**When to Use:**
- Use when optimizing MLPs for neural radiance fields or deformation fields, especially in few-shot or dynamic scenarios where high-frequency details are prone to noise [S1][S2, unofficial].
- Do not use when real-time inference is the primary concern and training time is not a bottleneck, as the annealing schedule adds complexity to the training loop [S1].

## Background
Originating from the observation that the number of frequencies $m$ in positional encoding controls the bandwidth of the Neural Tangent Kernel (NTK) [S1], this method was introduced by Park et al. [2020] and Hertz et al. [2021] to implement coarse-to-fine optimization strategies [S1]. It was adopted in Nerfies for optimizing deformation fields and in GeCoNeRF for few-shot neural radiance fields to suppress high-frequency occlusions [S1][S2, unofficial]. The technique builds on the work of Tancik et al. [2020] regarding Fourier positional encoding [S1][S2, unofficial].

## Sources
[S1] HyperNeRF, https://doi.org/10.1145/3478513.3480487, fetched 2026-10-28
[S2] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
