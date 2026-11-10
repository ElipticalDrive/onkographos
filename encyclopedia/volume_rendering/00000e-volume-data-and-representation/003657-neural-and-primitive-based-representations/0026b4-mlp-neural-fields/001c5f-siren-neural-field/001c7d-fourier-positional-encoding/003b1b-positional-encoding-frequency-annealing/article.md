# Positional Encoding Frequency Annealing

Also called: Fourier Frequency Annealing, Frequency Annealing

## Overview
Positional Encoding Frequency Annealing is a coarse-to-fine optimization strategy that constrains Fourier positional encoding to low dimensions initially to suppress high-frequency artifacts and self-occlusions [S1, unofficial]. It sets itself apart from standard Fourier Positional Encoding by dynamically controlling the bandwidth of the Neural Tangent Kernel (NTK) during training, preventing the network from overfitting to high-frequency noise or getting stuck in sub-optimal local minima caused by self-occlusions [S1, unofficial][S3].

## How it works
**Inputs and Outputs:**
- **Input:** Input coordinates $x$ (e.g., 3D spatial coordinates or ambient coordinates), number of frequency bands $m$, and an annealing parameter $\alpha$ (or iteration step $t$).
- **Output:** A higher-dimensional encoded vector where low-frequency components are active initially and high-frequency components are gradually introduced.

**Data Structures:**
- **Frequency Bands:** A set of frequencies defined as powers of 2. For $m$ bands, the frequencies are $2^0, 2^1, \dots, 2^{m-1}$ [S2, unofficial][S3].
- **Annealing Parameter:** A scalar value that increases over time, controlling the "window" of active frequencies [S3].

**Steps:**
1.  **Initialize Annealing Parameter:** Determine the value of the annealing parameter based on the optimization schedule.
    -   In the linear annealing variant [S1, unofficial], calculate $\alpha(t) = \frac{m \cdot t}{K}$, where $m$ is the number of encoding frequencies, $t$ is the current iteration step, and $K$ is a hyper-parameter.
    -   In the windowed variant [S3], $\alpha$ is a parameter that linearly increases from $0$ to $m$ over the course of training.
2.  **Compute Window/Mask:** Calculate a weight or mask for each frequency band index $j \in \{0, \dots, m-1\}$ based on the annealing parameter.
    -   **Linear Mask (Variant):** Use a clamped linear function. Let $levels = j$. The mask is $mask_j = \text{clamp}(progress \cdot m - levels, 0.0, 1.0)$, where $progress$ is derived from the annealing schedule (e.g., $t/K$) [S2, unofficial].
    -   **Window Function (Variant):** Use a truncated Hann window function: $w_j(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - j, 0, 1))}{2}$ [S3].
3.  **Apply Frequencies:** Multiply the input coordinates $x$ by the frequency bands to get scaled positions: $pts = x \cdot 2^j$ for each band $j$ [S2, unofficial].
4.  **Encode and Weight:** Compute the sine and cosine components for each band and apply the mask/window weights.
    -   $pts\_sin_j = \sin(pts_j) \cdot mask_j$ (or $w_j(\alpha)$)
    -   $pts\_cos_j = \cos(pts_j) \cdot mask_j$ (or $w_j(\alpha)$)
5.  **Concatenate:** Concatenate the weighted sine and cosine components along the feature dimension to produce the final encoded vector [S2, unofficial].

**Formulas:**
-   **Linear Annealing Equation:**
    $$ \alpha(t) = \frac{m \cdot t}{K} $$
    Where $m$ is the number of encoding frequencies, $t$ is the iteration step, and $K$ is a hyper-parameter [S1, unofficial].
-   **Window Function:**
    $$ w_j(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - j, 0, 1))}{2} $$
    Where $j \in \{0, \dots, m - 1\}$ is the frequency band index and $\alpha \in [0, m]$ is the annealing parameter [S3].
-   **Windowed Positional Encoding:**
    $$ \gamma_\alpha(x) = [w_0(\alpha)\sin(x), \dots, w_{m-1}(\alpha)\cos(2^{m-1}x)]^T $$
    [S3]

**Parameters:**
-   **$K$ (Hyper-parameter):** Controls the speed of annealing in the linear strategy. Set to 15,000 (15k) in reported experiments [S1, unofficial].
-   **$m$ (Number of Frequencies):** The total number of frequency bands used in the positional encoding.
-   **$\alpha$ (Annealing Parameter):** In the windowed variant, this parameter linearly increases from 0 to $m$ during optimization. It is equivalent to sliding a truncated Hann window down the frequency bands [S3].
-   **$\beta$ (Ambient Dimension Parameter):** In the windowed variant applied to ambient coordinates, $\beta$ controls the exposure of ambient dimensions. Setting $\beta=0$ disables identity concatenation and collapses ambient dimensions during initial optimization phases [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def linear_frequency_annealing(positions, freqs, iteration, K=15000):
    """
    Implements linear frequency annealing as described in [S1, unofficial] and [S2, unofficial].
    
    Args:
        positions: Tensor of input coordinates [..., D]
        freqs: Integer, number of frequency bands (m)
        iteration: Integer, current training step (t)
        K: Integer, hyper-parameter for annealing speed (default 15000)
        
    Returns:
        Encoded tensor [..., freqs * 2 * D]
    """
    # 1. Compute progress/alpha
    # alpha(t) = m * t / K
    # progress is effectively t/K, but we use the mask formula directly from [S2, unofficial]
    # mask = clamp(progress * freqs - levels, 0, 1)
    # Note: [S1, unofficial] defines alpha(t) = mt/K. [S2, unofficial] uses progress * freqs.
    # Assuming progress = t/K, then progress * freqs = (t/K) * freqs.
    # This matches the linear ramp logic.
    
    levels = torch.arange(freqs, device=positions.device)
    
    # Calculate the linear ramp value for each level
    # Using the formula from [S2, unofficial]: mask = clamp(progress * freqs - levels, 0, 1)
    # where progress = iteration / K
    progress = iteration / K
    ramp = progress * freqs
    
    # Compute mask for each frequency band
    mask = (ramp - levels).clamp(min=0.0, max=1.0)
    
    # 2. Compute frequency bands
    freq_bands = 2 ** levels
    
    # 3. Apply frequencies to positions
    # positions shape: [..., D]
    # freq_bands shape: [freqs]
    # pts shape: [..., D, freqs] (broadcasting)
    pts = positions[..., None] * freq_bands
    
    # 4. Apply sine/cosine and mask
    pts_sin = torch.sin(pts) * mask
    pts_cos = torch.cos(pts) * mask
    
    # 5. Concatenate and reshape
    # Concatenate along the last dimension
    pts_encoded = torch.cat([pts_sin, pts_cos], dim=-1)
    
    # Reshape to [..., freqs * 2 * D]
    # The last dimension of pts_encoded is freqs. 
    # We need to merge the freqs dimension with the D dimension? 
    # No, [S2, unofficial] says: reshape to (..., freqs * 2 * positions.shape[-1])
    # Let's trace [S2, unofficial] code:
    # pts_sin shape: [..., D, freqs]
    # pts_cos shape: [..., D, freqs]
    # cat dim=-1 -> [..., D, 2*freqs]
    # reshape to (..., freqs * 2 * D)
    # This implies the dimensions are merged.
    
    new_shape = positions.shape[:-1] + (freqs * 2 * positions.shape[-1],)
    pts_encoded = pts_encoded.reshape(new_shape)
    
    return pts_encoded

def windowed_frequency_annealing(positions, freqs, alpha):
    """
    Implements windowed frequency annealing as described in [S3].
    
    Args:
        positions: Tensor of input coordinates [..., D]
        freqs: Integer, number of frequency bands (m)
        alpha: Float, annealing parameter in [0, m]
        
    Returns:
        Encoded tensor [..., freqs * 2 * D]
    """
    levels = torch.arange(freqs, device=positions.device)
    
    # 1. Compute window function w_j(alpha)
    # w_j(alpha) = (1 - cos(pi * clamp(alpha - j, 0, 1))) / 2
    clamp_val = (alpha - levels).clamp(min=0.0, max=1.0)
    window = (1.0 - torch.cos(math.pi * clamp_val)) / 2.0
    
    # 2. Compute frequency bands
    freq_bands = 2 ** levels
    
    # 3. Apply frequencies to positions
    pts = positions[..., None] * freq_bands
    
    # 4. Apply sine/cosine and window weights
    pts_sin = torch.sin(pts) * window
    pts_cos = torch.cos(pts) * window
    
    # 5. Concatenate and reshape
    pts_encoded = torch.cat([pts_sin, pts_cos], dim=-1)
    
    new_shape = positions.shape[:-1] + (freqs * 2 * positions.shape[-1],)
    pts_encoded = pts_encoded.reshape(new_shape)
    
    return pts_encoded
```

**Edge Cases and Pitfalls:**
-   **Precision:** When $K$ is large and $t$ is small, the mask values may be very close to 0, potentially leading to vanishing gradients for high-frequency bands early in training.
-   **Boundary Conditions:** The `clamp` function ensures that the window/mask stays within $[0, 1]$. Failure to clamp can result in negative weights or values $>1$, distorting the frequency spectrum.
-   **Self-Occlusions:** Without annealing, high-frequency components can cause the network to fit noise or self-occlusions in the early stages, leading to poor convergence [S1, unofficial].

## Subtopics

### Positional Encoding with Linear Annealing
A positional encoding function that applies linear annealing to frequency bands based on a progress parameter, masking lower frequencies at progress=0 and exposing all at progress=1 [S2, unofficial]. It is chosen when a simple linear ramp of frequency exposure is sufficient to guide optimization from coarse to fine details. It differs from the windowed variant by using a hard linear mask rather than a smooth Hann window. Implementation involves creating frequency bands as powers of 2, computing a linear mask `clamp(progress * freqs - levels, 0, 1)`, and applying it to sine/cosine components [S2, unofficial].

### Windowed Positional Encoding
A technique to control the bandwidth of an MLP's Neural Tangent Kernel (NTK) by weighting frequency bands of positional encoding with a window function, enabling coarse-to-fine optimization to prevent overfitting and underfitting [S3]. It is chosen when smoother transitions between frequency bands are desired to avoid artifacts associated with hard masking. It sets itself apart by using a truncated Hann window $w_j(\alpha)$ that slides down the frequency bands as $\alpha$ increases from 0 to $m$. It is often applied to ambient dimensions in deformation fields, where setting $\beta=0$ can collapse ambient dimensions during initial optimization phases [S3].

## Operating characteristics
-   **Time Complexity:** The addition of the annealing mask/window calculation adds negligible overhead to the standard Fourier positional encoding, which is $O(m \cdot D)$ where $m$ is the number of frequencies and $D$ is the input dimension.
-   **Memory Complexity:** No additional memory is required beyond the standard positional encoding output.
-   **Performance:** Improves convergence in few-shot NeRF and deformation field optimization by suppressing high-frequency artifacts and self-occlusions [S1, unofficial]. Prevents the network from getting stuck in sub-optimal local minima [S3].
-   **Strengths:** Regularizes optimization, allowing the network to learn coarse structures before fine details. Effective for scenes with self-occlusions or limited input views.
-   **Limitations:** Requires careful tuning of hyper-parameters ($K$ or $\alpha$ schedule). If annealing is too slow, training may take longer to converge to high-frequency details. If too fast, it may not provide the regularization benefit.
-   **When to use:** When training neural fields on sparse views, dynamic scenes, or scenes with complex self-occlusions.
-   **When not to use:** When the scene is simple, well-lit, and has dense input views, standard Fourier encoding may suffice and annealing adds unnecessary complexity.

## Background
Positional Encoding Frequency Annealing originates from the need to stabilize the training of Neural Radiance Fields (NeRF) and related neural fields. The concept of using coarse-to-fine strategies in neural field optimization was introduced by Park et al. [2020] and Hertz et al. [2021] [S3]. Tancik et al. [2020] demonstrated that the number of frequencies in positional encoding controls the bandwidth of the Neural Tangent Kernel (NTK) [S3]. Building on this, Park et al. [2020] (Nerfies) used windowed positional encoding to optimize deformation fields in a coarse-to-fine manner [S3]. GeCoNeRF [S1, unofficial] adopted a linear annealing strategy to suppress high-frequency occlusions in few-shot NeRF. HyperNeRF [S3] further utilized windowed positional encoding for ambient dimensions to handle topological variations in dynamic scenes.

## Sources
[S1] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S3] HyperNeRF, https://doi.org/10.1145/3478513.3480487, fetched 2026-10-28
