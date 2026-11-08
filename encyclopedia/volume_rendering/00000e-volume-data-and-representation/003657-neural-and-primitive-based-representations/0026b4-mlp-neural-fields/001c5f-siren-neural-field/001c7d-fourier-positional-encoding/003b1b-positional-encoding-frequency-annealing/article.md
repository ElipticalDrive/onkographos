# Positional Encoding Frequency Annealing

Also called: Fourier Frequency Annealing, Frequency Annealing

## Overview
Positional Encoding Frequency Annealing is a coarse-to-fine optimization strategy that constrains Fourier positional encoding to low dimensions initially to suppress high-frequency artifacts and self-occlusions in neural radiance fields [S1, unofficial]. It sets itself apart from standard Fourier Positional Encoding by dynamically masking high-frequency bands during early training, forcing the network to prioritize low-frequency geometry before resolving fine details [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current training iteration step $t$, total annealing steps $K$, number of encoding frequencies $m$, and input coordinates.
- Output: Annealed positional encoding vector with high-frequency components masked out.

**Data Structures:**
- The positional encoding uses interleaved sine and cosine functions.
- The annealing factor $\alpha(t)$ is a scalar float.
- The mask is applied element-wise to the sine and cosine components.

**Steps:**
1. Calculate the annealing factor $\alpha(t)$ based on the current iteration $t$.
2. Determine the cutoff frequency based on $\alpha(t)$. Frequencies corresponding to indices greater than $\alpha(t) \times m$ are masked.
3. Compute the standard Fourier positional encoding for the input coordinates.
4. Apply the mask to the encoding vector, zeroing out high-frequency components.
5. Pass the masked encoding to the MLP.

**Formula:**
The annealing equation is defined as [S1, unofficial]:
$$ \alpha(t) = \frac{mt}{K} $$
Where:
- $m$ is the number of encoding frequencies.
- $t$ is the current iteration step.
- $K$ is a hyper-parameter controlling the duration of the annealing process.

The mask $M_i$ for the $i$-th frequency band (where $i$ ranges from $0$ to $m-1$) is:
$$ M_i = \begin{cases} 1 & \text{if } i < \alpha(t) \\ 0 & \text{otherwise} \end{cases} $$

**Parameters:**
- $K$: Set to 15,000 (15k) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def positional_encoding_frequency_annealing(positions, m, t, K):
    """
    Compute Fourier positional encoding with frequency annealing.
    
    Args:
        positions: Tensor of shape (..., D) containing input coordinates.
        m: Number of encoding frequencies (int).
        t: Current training iteration step (int).
        K: Hyper-parameter for annealing duration (int).
        
    Returns:
        Masked positional encoding tensor.
    """
    # Standard Fourier positional encoding setup
    # freq_bands = 2^0, 2^1, ..., 2^(m-1)
    levels = torch.arange(m, device=positions.device)
    freq_bands = 2 ** levels
    
    # Compute annealing factor alpha(t)
    # alpha(t) = m * t / K
    alpha = (m * t) / K
    
    # Determine which frequencies to keep
    # Keep frequencies where index < alpha
    # Mask is 1.0 if kept, 0.0 if masked
    # Note: alpha can exceed m, so we clamp or handle bounds
    mask = (levels < alpha).float()
    
    # Compute base frequencies
    # pts shape: (..., m)
    pts = positions[..., None] * freq_bands
    
    # Compute sine and cosine
    pts_sin = torch.sin(pts)
    pts_cos = torch.cos(pts)
    
    # Apply mask to both sin and cos components
    pts_sin = pts_sin * mask
    pts_cos = pts_cos * mask
    
    # Concatenate and reshape
    # Output shape: (..., m * 2 * D)
    pts_enc = torch.cat([pts_sin, pts_cos], dim=-1)
    
    # Reshape to match expected input dimension for MLP
    # Assuming positions has shape (..., D), output is (..., m*2*D)
    new_shape = list(positions.shape[:-1]) + [m * 2 * positions.shape[-1]]
    pts_enc = pts_enc.reshape(new_shape)
    
    return pts_enc
```

**Edge Cases and Pitfalls:**
- If $t=0$, $\alpha(0)=0$, and all frequencies are masked, resulting in a zero vector. This may cause initial training stagnation if not handled [G].
- If $t \ge K$, $\alpha(t) \ge m$, and all frequencies are exposed, effectively reverting to standard Fourier encoding.
- Precision issues may arise if $K$ is very large and $t$ is small, causing $\alpha(t)$ to be very close to 0.

## Subtopics

### Positional Encoding with Linear Annealing
A variant that applies linear annealing to frequency bands based on a progress parameter, masking lower frequencies at progress=0 and exposing all at progress=1 [S2, unofficial]. It creates frequency bands as powers of 2 ($2^{\text{levels}}$) and computes a mask using linear interpolation: $\text{mask} = \text{clamp}(\text{progress} \times \text{freqs} - \text{levels}, 0.0, 1)$ [S2, unofficial]. This allows for a smoother transition compared to the step-wise masking in the standard method. It is useful when a gradual exposure of frequencies is preferred over a sharp cutoff [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot m \cdot D)$, where $N$ is the number of points, $m$ is the number of frequencies, and $D$ is the input dimension. The annealing calculation is $O(m)$, which is negligible compared to the encoding computation.
- **Memory Complexity:** $O(N \cdot m \cdot D)$ for storing the encoded positions.
- **Performance:** Suppresses high-frequency artifacts and self-occlusions in the early stages of training, leading to more stable convergence [S1, unofficial].
- **Strengths:** Improves optimization stability for few-shot NeRF and complex scenes by preventing the network from fitting noise early on [S1, unofficial].
- **Limitations:** Requires careful tuning of $K$ and $m$. If $K$ is too small, the annealing finishes too quickly, losing its benefit. If too large, training may be slow to converge to fine details.
- **When to use:** When training neural radiance fields with limited data or complex geometry where high-frequency artifacts are problematic [S1, unofficial].
- **When not to use:** When real-time inference is critical and the overhead of dynamic masking is unacceptable, or when the scene is simple and does not suffer from high-frequency artifacts.

## Background
Originated in the context of few-shot Neural Radiance Fields (NeRF) to address high-frequency occlusions that fill the space between scene and camera [S1, unofficial]. The strategy was previously used by Park et al. (2021) and adopted in GeCoNeRF (2021) to regularize optimization by forcing the network to optimize coarse, low-frequency details first [S1, unofficial]. This eases the difficulty of the warping process in the beginning stages of training [S1, unofficial].

## Sources
[S1] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
