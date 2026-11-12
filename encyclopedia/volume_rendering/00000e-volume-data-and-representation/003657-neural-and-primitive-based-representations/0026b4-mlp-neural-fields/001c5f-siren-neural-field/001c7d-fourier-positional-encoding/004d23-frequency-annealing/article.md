# Frequency Annealing

Also called: BARF positional encoding, Frequency band windowing, Fourier Frequency Annealing, Annealed Positional Encoding, Coarse-to-fine Positional Encoding, Positional Encoding Frequency Annealing, Coarse-to-Fine Optimization via Positional Encoding Windowing

## Overview
Frequency Annealing is a training regularization strategy for Neural Radiance Fields (NeRF) that gradually activates higher-frequency bands of Fourier Positional Encoding [1267-fourier-positional-encoding.md] during optimization. It solves the problem of optimization instability and local minima caused by high-frequency details appearing too early in training, which can lead to artifacts like self-occlusions or incorrect geometry [S1, S4, unofficial]. Unlike standard Fourier Feature Networks [2765-fourier-feature-networks.md] which use all frequencies from the start, Frequency Annealing employs a coarse-to-fine schedule, effectively narrowing the Neural Tangent Kernel (NTK) bandwidth initially to prioritize low-frequency structure before refining high-frequency details [S2, S6].

## How it works
Frequency Annealing modifies the input to the MLP by applying a time-dependent window function to the frequency bands of the positional encoding. This ensures that low-frequency components dominate the gradient updates early in training, while high-frequency components are gradually introduced.

### Inputs and Outputs
- **Inputs**:
  - Input coordinates $x \in \mathbb{R}^3$ (or $\mathbb{R}^2$ for 2D embeddings).
  - Training iteration $t$ or progress variable $\rho$.
  - Hyperparameters: $L$ (total number of frequency bases), $K$ (annealing duration in iterations), $m$ (maximum band index, often equal to $L$).
- **Outputs**:
  - Annealed positional encoding vector $\gamma_\alpha(x)$ or $\gamma(x; \rho)$.

### Data Structures
The positional encoding is represented as a concatenation of weighted sine and cosine pairs.
- **Base Encoding**: $\gamma(x) = [x, \gamma_0(x), \gamma_1(x), \dots, \gamma_{L-1}(x)] \in \mathbb{R}^{3+6L}$ [S1, unofficial].
- **Band Encoding**: Each band $k$ produces a 6-dimensional vector (for 3D input): $\gamma_k(x) = [\cos(2^k \pi x), \sin(2^k \pi x)] \in \mathbb{R}^6$ [S1, unofficial].
- **Annealed Band**: $\gamma_k(x; \rho) = w_k(\rho) \cdot [\cos(2^k \pi x), \sin(2^k \pi x)]$ [S1, unofficial].

### Numbered Steps
1. **Initialize Parameters**: Set the total number of frequency bases $L$ (e.g., 10 for 3D points, 4 for viewing directions) [S1, unofficial]. Define the annealing schedule range, typically from iteration 0 to $K$ (e.g., 15,000 iterations) [S4, unofficial].
2. **Compute Progress Variable**: Calculate the progress variable $\rho$ (or $\alpha$) based on the current training iteration $t$.
   - In the BARF/UP-NeRF formulation, $\rho$ linearly increases from 0 to $L$ over the first 10% to 50% of total training iterations [S1, unofficial].
   - In the HyperNeRF/Nerfies formulation, $\alpha(t) = m \cdot t / K$, where $m$ is the number of encoding frequencies and $K$ is a hyperparameter (e.g., 15k) [S4, unofficial].
3. **Apply Window Function**: For each frequency band index $k \in \{0, \dots, L-1\}$, compute the weight $w_k(\rho)$ using a smooth window function (typically a truncated Hann window) [S2, S6].
4. **Construct Encoding**: Multiply the base sine/cosine values of each band by its corresponding weight $w_k(\rho)$ and concatenate them with the raw coordinates $x$ [S1, unofficial][S6].
5. **Feed to MLP**: Pass the resulting vector $\gamma(x; \rho)$ into the deformation or color MLP [S2].

### Formulas
The windowing function $w_k(\rho)$ determines the activation of band $k$ based on progress $\rho$ [S1, unofficial]:

$$
w_k(\rho) = 
\begin{cases} 
0 & \text{if } \rho < k \\
1 - \frac{\cos((\rho - k)\pi)}{2} & \text{if } 0 \le \rho - k < 1 \\
1 & \text{if } \rho - k \ge 1 
\end{cases}
$$

[S1, unofficial]

Alternatively, using the clamp notation common in implementations [S6]:

$$
w_j(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - j, 0, 1))}{2}
$$

where $j$ is the band index and $\alpha$ is the annealing parameter [S6].

The final annealed encoding is:

$$
\gamma_\alpha(x) = [x, w_0(\alpha)\gamma_0(x), w_1(\alpha)\gamma_1(x), \dots, w_{L-1}(\alpha)\gamma_{L-1}(x)]
$$

[S1, unofficial][S6]

### Parameters
- **$L$ (or $m$)**: Number of frequency bases. Typical value: 10 for 3D point $x$, 4 for viewing direction $d$ [S1, unofficial].
- **$\rho$ / $\alpha$**: Scheduling variable. $\rho$ goes from 0 to $L$ [S1, unofficial]. $\alpha$ goes from 0 to $m$ [S6].
- **$K$**: Hyperparameter controlling the duration of annealing. Set to 15,000 iterations in some implementations [S4, unofficial].
- **Near/Far Clipping**: For coarse model sampling, points are sampled between 0.1 and 5.0 [S1, unofficial].

### Pseudocode
The following Python-like pseudocode implements the windowed positional encoding described in [S1, unofficial] and [S6]. It assumes `positions` is a tensor of shape `[..., 3]` and `freqs` is $L$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def frequency_annealing_encoding(positions, freqs, progress):
    """
    Computes frequency-annealed positional encoding.
    
    Args:
        positions: Tensor of shape [..., 3] containing input coordinates.
        freqs: Integer L, the number of frequency bands.
        progress: Float between 0 and freqs. Represents the current annealing state.
                  Equivalent to rho in [S1, unofficial] or alpha in [S6].
    
    Returns:
        Tensor of shape [..., 3 + 6*freqs] containing the annealed encoding.
    """
    # 1. Generate frequency bands: 2^0, 2^1, ..., 2^(freqs-1)
    # Using integer range for levels
    levels = list(range(freqs))
    freq_bands = [2**k for k in levels]
    
    # 2. Compute window weights w_k(progress)
    # Formula from [S1, unofficial]:
    # w_k = 0 if progress < k
    # w_k = 1 - cos((progress - k)*pi)/2 if 0 <= progress - k < 1
    # w_k = 1 if progress - k >= 1
    
    weights = []
    for k in levels:
        diff = progress - k
        if diff < 0:
            w = 0.0
        elif diff < 1:
            w = 1.0 - math.cos(diff * math.pi) / 2.0
        else:
            w = 1.0
        weights.append(w)
        
    # 3. Apply weights to sine and cosine components
    # Base encoding for band k: [cos(2^k * pi * x), sin(2^k * pi * x)]
    # We need to broadcast positions and freq_bands correctly.
    
    # Initialize output list
    encoded_components = []
    
    # Add raw coordinates (identity)
    # Note: Some implementations omit identity for ambient coords [S6], 
    # but standard NeRF/BARF includes it [S1, unofficial]. We include it here as per [S1, unofficial].
    encoded_components.append(positions)
    
    # Process each frequency band
    for k in range(freqs):
        freq = freq_bands[k]
        w = weights[k]
        
        # Compute argument: 2^k * pi * x
        # positions shape: [..., 3]
        # freq is scalar
        arg = positions * freq * math.pi
        
        # Compute sin and cos
        sin_vals = math.sin(arg) # Element-wise sin
        cos_vals = math.cos(arg) # Element-wise cos
        
        # Apply weight
        # If w is 0, these become zero tensors
        weighted_sin = sin_vals * w
        weighted_cos = cos_vals * w
        
        # Concatenate sin and cos for this band
        # Result shape: [..., 6]
        band_encoding = concatenate([weighted_sin, weighted_cos], axis=-1)
        encoded_components.append(band_encoding)
        
    # 4. Concatenate all components
    # Final shape: [..., 3 + 6*freqs]
    result = concatenate(encoded_components, axis=-1)
    
    return result
```

### Edge Cases and Pitfalls
- **Zero Progress**: When $\rho=0$ or $\alpha=0$, all frequency bands are weighted to 0. If the identity mapping (raw coordinates) is not included, the input to the MLP becomes zero, potentially causing NaNs or stagnation [S2, S6].
- **Large Motions**: Disabling coarse-to-fine regularization causes significant performance drops in dynamic scenes with large motions due to local minima [S2].
- **Topological Changes**: Frequency annealing alone may fail if the scene undergoes topological changes (e.g., objects appearing/disappearing), as the deformation field assumptions may be violated [S2].

## Subtopics

### Positional Encoding with Linear Annealing
A variant that uses a linear mask instead of a smooth window function. It computes a mask `clamp(progress * freqs - levels, 0, 1)` and multiplies the sine/cosine components directly. This is computationally simpler but lacks the smooth gradient transition of the Hann window [S5, unofficial].

### Windowed Positional Encoding
Used in HyperNeRF [S6], this variant applies the windowing specifically to ambient dimensions (e.g., slicing surfaces) while keeping spatial coordinates unwindowed or using a separate schedule. It allows collapsing ambient dimensions by setting $\beta=0$ initially, preventing overfitting in the ambient space during early optimization [S6].

## Operating characteristics
- **Time Complexity**: Adds negligible overhead to the forward pass, as it involves simple multiplication of weights against precomputed sine/cosine values. The annealing schedule is computed once per iteration or batch.
- **Memory Complexity**: No additional memory storage is required beyond the standard positional encoding buffer.
- **Performance**: Significantly improves convergence stability and final reconstruction quality, especially for few-shot or dynamic scenes. On DTU with 3 views, CF PE (IV) yields PSNR 18.30 (21.01) and SSIM 0.78 (0.87), compared to Full PE (I) which yields PSNR 16.27 (18.41) and SSIM 0.69 (0.81) [S3].
- **Strengths**: Prevents high-frequency artifacts early in training; stabilizes optimization for deformable scenes; reduces local minima.
- **Limitations**: Requires careful tuning of the annealing schedule ($K$, start/end iterations); may slow down initial convergence if the schedule is too slow [S4, unofficial].

## Background
Frequency Annealing was introduced in the context of NeRF optimization to address the difficulty of optimizing high-frequency details from sparse views. It was first popularized by Park et al. in the BARF (Blind Optimization of Radiance Fields) paper [23] (cited in [S1, unofficial]) and further detailed in Nerfies [S2] and HyperNeRF [S6]. The method leverages the property that the Neural Tangent Kernel (NTK) bandwidth is controlled by the highest frequency in the positional encoding [S2, S6]. By gradually increasing the maximum active frequency, the optimization landscape is smoothed initially, allowing the network to learn coarse geometry before refining details.

## Sources
[S1] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
[S2] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
[S3] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S4] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S5] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S6] HyperNeRF, https://doi.org/10.1145/3478513.3480487, fetched 2026-10-28
