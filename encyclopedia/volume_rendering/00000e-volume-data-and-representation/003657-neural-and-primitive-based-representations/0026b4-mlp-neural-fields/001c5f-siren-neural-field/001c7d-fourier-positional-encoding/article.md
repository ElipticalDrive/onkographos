# Fourier Positional Encoding

Also called: 1D Positional Embedding, Fourier Feature Embedding, Fourier Feature Mappings, Fourier Feature Positional Encoding, Fourier Features, Sinusoidal Embedding, Sinusoidal Encoding, Sinusoidal Positional Encoding

## Overview
Fourier Positional Encoding (FPE) maps low-dimensional input coordinates (such as 3D positions or 2D view directions) into a higher-dimensional space using a multi-resolution sequence of sine and cosine functions at exponentially increasing frequencies [S1, S2, S14, unofficial]. It solves the problem of spectral bias in standard Multi-Layer Perceptrons (MLPs), which inherently prioritize learning low-frequency signals and fail to capture high-frequency scene details [S6][S19, unofficial]. Unlike its siblings, it provides a fixed, high-dimensional basis that allows the subsequent MLP to learn fine spatial variations without needing to discover the frequency spectrum itself [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A tensor of coordinates $x \in \mathbb{R}^{B \times D}$, where $B$ is the batch size and $D$ is the input dimensionality (typically $D=3$ for position, $D=3$ for view direction).
- Output: A tensor $z \in \mathbb{R}^{B \times D_{out}}$, where $D_{out}$ is the encoded feature dimension.

**Data Structures and Parameters:**
- `num_freqs` (or `num_octaves`, `n_harmonic_functions`): Integer $N$, the number of frequency bands. Default is often 10 for position, 4 for direction [S11, S20, unofficial].
- `log_sampling` (or `logspace`): Boolean. If True, frequencies are spaced logarithmically ($2^i$); if False, linearly [S3, S4, unofficial]. Default is True [S4, S5, unofficial].
- `include_input`: Boolean. If True, the raw input $x$ is concatenated to the encoded output [S4, S14, unofficial]. Default is True [S4, S18, unofficial].
- `omega0`: Float. A scaling factor for frequencies. Default is 1.0 [S3, S7, unofficial].
- `max_freq_log2`: Integer. The maximum frequency exponent. Often set to `num_freqs - 1` [S20, unofficial].

**Formulas:**
The frequency bands $f_k$ for $k \in \{0, \dots, N-1\}$ are generated as:
$$
f_k = \omega_0 \cdot 2^{\frac{k \cdot \text{max\_freq\_log2}}{N-1}} \quad \text{(if log\_sampling is False, linear in exponent)}
$$
Or more commonly in implementation (powers of 2):
$$
f_k = \omega_0 \cdot 2^k \quad \text{(if log\_sampling is True)}
$$
The encoding for a single scalar input $x$ with $N$ harmonics is:
$$
\gamma(x) = (\sin(2^0 \pi x), \cos(2^0 \pi x), \dots, \sin(2^{N-1} \pi x), \cos(2^{N-1} \pi x))
$$
If `include_input` is True, the output is concatenated with $x$ [S11, S14, unofficial].

The output dimension $D_{out}$ is:
$$
D_{out} = D \times (2 \times N + \text{include\_input})
$$
For $D=3, N=10, \text{include\_input}=\text{True}$, $D_{out} = 3 \times (20 + 1) = 63$ [S8, S11, unofficial].

**Steps:**
1. Generate frequency bands $f_k$ based on `log_sampling` and `max_freq_log2`.
2. Scale the input coordinates $x$ by the frequencies: $e = x \cdot f_k$.
3. Apply sine and cosine functions to the scaled inputs: $\sin(e)$ and $\cos(e)$.
4. Concatenate the sine and cosine results along the feature dimension.
5. If `include_input` is True, concatenate the original input $x$ to the result.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def fourier_positional_encoding(x, num_freqs=10, log_sampling=True, include_input=True, omega0=1.0):
    """
    x: Tensor of shape (*, D)
    Returns: Tensor of shape (*, D * (2 * num_freqs + include_input))
    """
    # 1. Generate frequency bands
    if log_sampling:
        # Powers of 2: 2^0, 2^1, ..., 2^(num_freqs-1)
        freq_bands = 2.0 ** torch.arange(num_freqs, device=x.device, dtype=x.dtype)
    else:
        # Linear spacing from 2^0 to 2^(num_freqs-1)
        freq_bands = torch.linspace(2.0**0, 2.0**(num_freqs - 1), num_freqs, device=x.device, dtype=x.dtype)
    
    # Scale by omega0
    freq_bands = omega0 * freq_bands
    
    # 2. Reshape for broadcasting: x is (*, D), freq_bands is (num_freqs,)
    # We want to compute sin(x * freq) for each freq
    # x[..., None] shape: (*, D, 1)
    # freq_bands[None, :] shape: (1, num_freqs) -> broadcast to (*, D, num_freqs)
    
    # Expand x to (*, D, 1)
    x_expanded = x.unsqueeze(-1)
    
    # Compute scaled inputs: (*, D, num_freqs)
    scaled_inputs = x_expanded * freq_bands
    
    # 3. Apply sine and cosine
    # sin_part shape: (*, D, num_freqs)
    sin_part = torch.sin(scaled_inputs)
    cos_part = torch.cos(scaled_inputs)
    
    # 4. Concatenate sin and cos along the last dimension
    # Result shape: (*, D, 2 * num_freqs)
    encoded = torch.cat([sin_part, cos_part], dim=-1)
    
    # 5. If include_input, concatenate original x
    if include_input:
        # encoded shape: (*, D, 2 * num_freqs)
        # x shape: (*, D)
        # We need to align dimensions. x is (*, D), encoded is (*, D, 2*N)
        # Reshape x to (*, D, 1) then cat? No, we want to append the D dimensions.
        # Actually, the standard layout interleaves or blocks. 
        # The formula implies: [sin(2^0 x), cos(2^0 x), ..., sin(2^N x), cos(2^N x)]
        # This is per dimension.
        # Let's reshape encoded to (*, D * 2 * num_freqs)
        # And x to (*, D)
        
        # Flatten the last two dims of encoded: (*, D * 2 * num_freqs)
        encoded_flat = encoded.reshape(*x.shape[:-1], -1)
        
        # Concatenate x (which is (*, D)) to the end
        # Note: Some implementations prepend x. The formula in S11 shows x first.
        # S14 shows x first.
        result = torch.cat([x, encoded_flat], dim=-1)
    else:
        result = encoded.reshape(*x.shape[:-1], -1)
        
    return result
```

**Edge Cases and Pitfalls:**
- **Lambda Closure Issue:** When implementing FPE as a list of lambda functions (common in TensorFlow/PyTorch modules), variables `p_fn` and `freq` must be bound as default arguments in the loop (e.g., `lambda x, p_fn=p_fn, freq=freq: p_fn(x * freq)`) to avoid capturing the final loop values [S18, unofficial].
- **Spectral Bias:** Without FPE, ReLU MLPs struggle to represent high-frequency details, leading to blurry reconstructions [S6].
- **Gradient Locality:** Unlike hash encodings, FPE derivatives are continuous across space, allowing for stable second-order derivative computations (e.g., for curvature regularization) [S12].

## Subtopics

### [Fourier Feature Networks](2765-fourier-feature-networks.md)
Fourier Feature Networks (FFN) address the spectral bias of ReLU MLPs by replacing the network's internal featurizer with a fixed, high-dimensional Fourier embedding. It allows the network to predict signals (1D, 2D, or 3D radiance fields) more effectively by introducing higher-frequency information directly into the input. Choose this when using standard ReLU MLPs that fail to converge on high-frequency details. [Fourier Feature Networks](2765-fourier-feature-networks.md)

### [Off-Axis Positional Encoding](3503-off-axis-positional-encoding.md)
Off-Axis Positional Encoding uses a large projection matrix $P$ derived from a twice-tessellated icosahedron to encode anisotropic Gaussians, disambiguating shapes with identical marginal distributions along basis axes. It differs from standard FPE by capturing off-axis correlations, enabling accurate Kalman-like warping for highly anisotropic primitives. Choose this for 3D Gaussian Splatting variants with strong anisotropy. [Off-Axis Positional Encoding](3503-off-axis-positional-encoding.md)

### [Coordinate-Time Encoding](5105-coordinate-time-encoding.md)
Coordinate-Time Encoding (One-Blob Encoding) is a continuous, single-scale positional encoding for bounded domains. It provides a localized representation by rasterizing a kernel, achieving higher accuracy in bounded regions at the cost of lacking multi-scale frequency support. Choose this for dynamic in vivo sequences where spatiotemporal features need concatenation. [Coordinate-Time Encoding](5105-coordinate-time-encoding.md)

### Integrated Positional Encoding (Diagonal Covariance)
Integrated Positional Encoding (IPE) generalizes FPE to encode a region of space (e.g., a cone) rather than a single point, solving aliasing artifacts in NeRFs [S1, unofficial]. It computes the expected value of sine/cosine features over an interval, shrinking high-frequency features towards zero when the frequency period is small compared to the interval size [S1, unofficial]. Choose this when rendering at multiple resolutions or with large sample volumes. [Integrated Positional Encoding](5255-integrated-positional-encoding.md)

### [Positional Encoding Frequency Annealing](8902-positional-encoding-frequency-annealing.md)
Positional Encoding Frequency Annealing is a coarse-to-fine strategy that constrains FPE to low dimensions initially to suppress high-frequency artifacts and self-occlusions. It dynamically controls the bandwidth of the Neural Tangent Kernel (NTK) during training to prevent overfitting to noise. Choose this when training is unstable or stuck in local minima due to self-occlusions. [Positional Encoding Frequency Annealing](8902-positional-encoding-frequency-annealing.md)

### [2D Positional Embedding](12829-2d-positional-embedding.md)
2D Positional Embedding maps 2D spatial coordinates $(x, y)$ into a higher-dimensional space using interleaved sine and cosine functions. It is explicitly designed for 2D inputs, such as image feature maps, within neural fields. Choose this for 2D image processing contexts rather than 3D volumetric rendering. [2D Positional Embedding](12829-2d-positional-embedding.md)

### Scale Featurization
Scale Featurization provides scale information to the MLP by averaging and concatenating a featurized version of downweighting factors, ensuring consistent feature magnitude [S25]. It uses an approximation for $\operatorname{erf}(x)$ for speed and applies a stop-gradient operator to the mean of squared grid values [S25]. Choose this in grid-based methods like Zip-NeRF to maintain scale consistency.

### Conditional Positional Encoding
Conditional Positional Encoding (CPE) applies phase shifts based on a time index to sampled features, inspired by Fourier transformation phase retention [S26]. It is applied to 6D attributes and texture features but not density fields to maintain zero initialization [S26]. Choose this for 4D novel view synthesis (V4D) where temporal dynamics are encoded.

### Integrated Positional Encoding (Diagonal Covariance)
This variant estimates the mean of $\sin(z)$ where $z$ is a Gaussian random variable, accounting for Gaussian covariance uncertainty [S16, unofficial]. It computes the expected value of sine/cosine functions under the Gaussian distribution, using an exponential decay term $\exp(-0.5 \cdot x_{var})$ to shrink sine values when variance is wide [S16, unofficial]. Choose this when using 3D Gaussians with diagonal covariance matrices in integrated encoding.

## Operating characteristics
- **Time Complexity:** $O(D \cdot N)$ per sample, where $D$ is input dimension and $N$ is number of frequencies. The operation is vectorizable and fast [S2, S4, unofficial].
- **Memory Complexity:** $O(D \cdot N)$ for the output tensor. The encoding itself is stateless and requires no stored parameters if frequencies are fixed [S1, S19, unofficial].
- **Performance:** Requires a larger network (e.g., 8 hidden layers, 256 wide) compared to hash encoding to achieve similar quality, and trains slower [S1, unofficial][S22]. However, it is significantly faster to train than learned positional embeddings because the encoder is fixed [S1, unofficial].
- **Strengths:** Continuous derivatives allow for curvature regularization [S12]; simple to implement; effective at overcoming spectral bias [S6].
- **Limitations:** Slower training than sparse hash encodings; requires careful tuning of `num_freqs` [S1, unofficial][S22].
- **When to use:** When using standard MLPs (ReLU/SIREN) and high-frequency detail is critical, and training time is not the primary constraint.
- **When not to use:** When real-time inference or very fast training is required; consider Hash Encoding or Grid Encoding instead [S1, unofficial][S22].

## Background
Fourier Positional Encoding was adopted in computer graphics by Mildenhall et al. in the NeRF paper (ECCV 2020) to encode the spatiodirectionally varying light field and volume density [S1, S8, unofficial]. It was inspired by positional encodings used in natural language processing (Vaswani et al., 2017) [S1, unofficial][S6]. The method was later extended to randomly oriented parallel wavefronts (Tancik et al., 2020) and level-of-detail filtering (Barron et al., 2021a) [S1, unofficial]. It is widely used in SIREN-based neural fields and various NeRF variants [S1, S19, unofficial].

## Sources
[S1] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
[S2] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S3] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S4] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S5] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S6] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S7] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S8] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S9] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S10] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S11] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S12] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S13] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S14] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S15] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S16] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S17] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S18] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S19] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
[S20] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S21] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
[S22] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S23] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S24] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S25] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S26] V4D: Voxel for 4D Novel View Synthesis, https://doi.org/10.1109/tvcg.2023.3312127, fetched 2026-10-26
