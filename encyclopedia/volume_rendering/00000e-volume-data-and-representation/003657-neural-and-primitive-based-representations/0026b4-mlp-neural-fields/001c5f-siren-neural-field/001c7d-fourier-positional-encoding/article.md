# Fourier Positional Encoding

Also called: Fourier Features, Sinusoidal Encoding, Sinusoidal Embedding, Fourier Feature Mappings, Fourier Feature Embedding, Sinusoidal Positional Encoding, Fourier Feature Positional Encoding, Positional Encoding (Mildenhall et al, 2020)

## Overview
Fourier Positional Encoding (FPE) maps low-dimensional input coordinates (such as 3D positions or 2D viewing directions) into a higher-dimensional feature space using a multi-resolution sequence of sine and cosine functions at exponentially increasing frequencies [S1][S2, S3, S4, S5, S6, unofficial][S7][S8, S9, S10, S11, S12, unofficial][S13][S14, unofficial][S15][S16, S17, S18, S19, S20, S21, S22, S23, unofficial][S24, unofficial][S25, unofficial][S26][S27, unofficial][S28][S29, S30, unofficial]. It solves the problem of spectral bias in neural networks, where standard MLPs with ReLU activations struggle to learn high-frequency details because they prioritize smooth, low-frequency signals [S7][S21, unofficial]. By lifting inputs to a Fourier space, FPE allows the MLP to capture fine-scale variations without needing to learn the frequency basis itself [S1][S6, unofficial][S7][S9, S12, S16, S19, S20, S22, unofficial]. It sets itself apart from learned feature mappings (like [Fourier Feature Networks](2765-fourier-feature-networks.md)) by using fixed, predefined frequencies, and from hash-based encodings by providing continuous derivatives suitable for higher-order computations [S13, S26].

## How it works
**Inputs and Outputs:**
- Input: A tensor of coordinates $x \in \mathbb{R}^{d_{in}}$ (e.g., $d_{in}=3$ for 3D position, $d_{in}=2$ for 2D direction) [S1, S7][S9, S12, unofficial][S15, S28].
- Output: A tensor of encoded features $z \in \mathbb{R}^{d_{out}}$ [S1, S7][S9, S12, unofficial][S15, S28].

**Data Structures and Parameters:**
- `input_dims` ($d_{in}$): Dimensionality of the input vector [S5, S8, S21, S23, unofficial].
- `num_freqs` ($N$ or $L$): Number of frequency bands (octaves/harmonics). Typical value: 10 for position, 4 for direction [S9, S12, unofficial][S28].
- `log_sampling` (bool): Whether frequencies are exponentially spaced (True) or linearly spaced (False). Default: True [S4, S5, S6, S8, S10, unofficial][S15][S20, S22, unofficial].
- `max_freq_log2` ($M$): The maximum frequency exponent. If `log_sampling` is True, the highest frequency is $2^M$. Typically $M = N-1$ [S6, unofficial][S15][S20, S22, unofficial].
- `include_input` (bool): Whether to concatenate the original input coordinates to the embedding. Default: True [S5, S6, S8, S10, unofficial][S15][S20, S22, unofficial].
- `omega0` (float): A scaling factor for frequencies. Default: 1.0 [S4, S8, unofficial].

**Formulas:**
The frequency bands $f_k$ are generated based on the sampling strategy:
- Logarithmic (default): $f_k = 2^k$ for $k \in [0, N-1]$ [S4, S5, S6, S8, S10, unofficial][S15][S16, S18, S27, S29, S30, unofficial].
- Linear: $f_k$ is linearly spaced between $2^0$ and $2^{N-1}$ [S4, S5, S6, S8, S10, unofficial][S15][S20, S22, unofficial].

The encoding for a scalar input $p$ at level $l$ is:
$$ \gamma_l(p) = (\sin(2^l \pi p), \cos(2^l \pi p)) $$
[S1][S9, S11, unofficial][S13][S24, unofficial]

The full encoding for a vector $x$ is computed by applying this to each dimension independently and concatenating the results [S1][S18, S20, S22, unofficial]. If `include_input` is True, the original $x$ is prepended [S5, S6, S8, S10, unofficial][S15][S19, S25, unofficial].

**Output Dimension:**
$$ d_{out} = d_{in} \times (2 \times N + \text{include\_input}) $$
[S4, S8, S10, S20, unofficial]

**Numbered Steps:**
1. Generate frequency bands $f_k$ for $k=0 \dots N-1$ using `log_sampling` and `max_freq_log2` [S4, S5, S6, S8, S10, unofficial][S15][S20, S22, unofficial].
2. Scale frequencies by `omega0` (if applicable) [S4, S8, unofficial].
3. Initialize an empty list `parts` or output tensor [S5, S16, S20, S21, unofficial].
4. If `include_input` is True, append the original input $x$ to `parts` [S5, S6, S8, S10, unofficial][S15][S19, S25, unofficial].
5. For each frequency $f_k$ in the generated bands:
    a. Compute scaled input: $e = x \cdot f_k$ [S8, S10, S27, S29, unofficial].
    b. Compute sine and cosine components: $\sin(e)$ and $\cos(e)$ [S3, S8, unofficial][S15][S19, S25, S27, S29, unofficial].
    c. Append $\sin(e)$ and $\cos(e)$ to `parts` [S8, unofficial][S15][S19, S25, unofficial].
6. Concatenate all elements in `parts` along the last dimension to produce the final embedding $z$ [S15][S16, S18, S20, S22, S25, S27, S29, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def fourier_positional_encoding(x, input_dims, num_freqs, max_freq_log2, log_sampling=True, include_input=True, omega0=1.0):
    """
    Computes Fourier Positional Encoding for input coordinates.
    
    Args:
        x: torch.Tensor of shape (*, input_dims)
        input_dims: int, dimensionality of input (e.g., 3)
        num_freqs: int, number of frequency bands (N)
        max_freq_log2: int, log2 of max frequency (M)
        log_sampling: bool, if True use exponential spacing, else linear
        include_input: bool, if True prepend original x
        omega0: float, frequency scaling factor
        
    Returns:
        torch.Tensor of shape (*, output_dim)
    """
    # Step 1: Generate frequency bands
    if log_sampling:
        # Exponential spacing: 2^0, 2^1, ..., 2^(N-1)
        freq_bands = 2.0 ** torch.linspace(0, max_freq_log2, num_freqs, device=x.device, dtype=x.dtype)
    else:
        # Linear spacing from 2^0 to 2^(N-1)
        freq_bands = torch.linspace(2.0**0, 2.0**max_freq_log2, num_freqs, device=x.device, dtype=x.dtype)
        
    # Step 2: Scale frequencies
    freq_bands = freq_bands * omega0
    
    # Step 3: Prepare parts list
    parts = []
    if include_input:
        parts.append(x)
        
    # Step 4: Compute sine and cosine for each frequency
    # x shape: (*, input_dims)
    # freq_bands shape: (num_freqs,)
    # Broadcasting produces (*, input_dims, num_freqs)
    scaled_x = x[..., None] * freq_bands 
    
    # Compute sin and cos
    sin_vals = torch.sin(scaled_x)
    cos_vals = torch.cos(scaled_x)
    
    # Reshape to (*, input_dims * num_freqs)
    sin_vals = sin_vals.view(*x.shape[:-1], -1)
    cos_vals = cos_vals.view(*x.shape[:-1], -1)
    
    parts.append(sin_vals)
    parts.append(cos_vals)
    
    # Step 5: Concatenate
    return torch.cat(parts, dim=-1)
```

**Edge Cases and Pitfalls:**
- **Lambda Closure Issue:** When implementing FPE using a list of lambda functions (common in TensorFlow/PyTorch dynamic graphs), variables `p_fn` and `freq` must be bound explicitly as default arguments (e.g., `lambda x, p_fn=p_fn, freq=freq: p_fn(x * freq)`) to avoid capturing the final loop values [S20, unofficial].
- **Precision:** Directly regressing from raw coordinates fails due to spectral bias; FPE mitigates this but requires sufficient `num_freqs` to capture scene details [S7][S21, unofficial].
- **Boundary Conditions:** The encoding assumes inputs are in a reasonable range. If inputs are not normalized, high frequencies may alias or overflow [G].

## Subtopics

### [Fourier Feature Networks](2765-fourier-feature-networks.md)
Remedies spectral bias in ReLU MLPs by replacing the learning burden of a featurizer with a fixed, high-dimensional Fourier embedding.

### [Off-Axis Positional Encoding](3503-off-axis-positional-encoding.md)
Disambiguates anisotropic Gaussians with identical marginal distributions by using a non-identity projection matrix derived from an icosahedron.

### [Integrated Positional Encoding](5255-integrated-positional-encoding.md)
Anti-aliases high-frequency details by computing the closed-form integral of sine/cosine features over a Gaussian approximation of the camera frustum.

### [Integrated Positional Encoding (Diagonal Covariance)](12189-integrated-positional-encoding-diagonal-covariance.md)
Suppresses high-frequency components when spatial variance is large by computing the expected value of sinusoidal encodings under Gaussian uncertainty.

### [2D Positional Embedding](12829-2d-positional-embedding.md)
Applies fixed, predefined frequencies to 2D grid or batched coordinate inputs common in vision tasks, unlike the learned frequencies of Fourier Feature Networks.

### [Frequency Annealing](13443-frequency-annealing.md)
Narrows the Neural Tangent Kernel bandwidth initially by gradually activating higher-frequency bands during optimization to avoid local minima.

### [Truncated Dynamic Low-pass Filter](14016-truncated-dynamic-low-pass-filter.md)
Prevents convergence to local optima in pose estimation by initializing the frequency threshold at a non-zero value, ensuring valid color information from the first iteration.

### [Rotary Position Embedding (RoPE)](15851-rotary-position-embedding-rope.md)
Injects spatial position via relative rotational transformations that preserve token inner products, rather than mapping to a fixed trigonometric basis.

### [Conditional and Temporal Positional Encoding](17025-conditional-and-temporal-positional-encoding.md)
Extends static encodings to dynamic scenes by introducing time-dependent phase shifts or explicit spatiotemporal concatenation.

### [Tri-Plane Positional Encoding](17482-tri-plane-positional-encoding.md)
Suppresses high-frequency noise in grid-based features by expanding the implicit function into trigonometric terms multiplied by 2D plane features.

### [Self-Attention Convolutions](17500-self-attention-convolutions.md)
Addresses discretization discontinuities by dynamically producing multi-frequency feature maps via self-attention mechanisms with varying window sizes.

### [Intrinsic Neural Fields Training Pipeline](18393-intrinsic-neural-fields-training-pipeline.md)
Embeds surface points into Laplace-Beltrami eigenfunctions to adapt the representation to the manifold's intrinsic geometry rather than using fixed frequencies.

### Scale Featurization
A method to provide scale information to the MLP by averaging and concatenating a featurized version of the downweighting factors $\omega_{j,\ell}$, ensuring features have consistent magnitude regardless of training progression [S34]. It scales the result by the standard deviation of values in $V_\ell$, padded with $\initval$ to prevent zero values, and applies a stop-gradient operator to the mean of the squared grid values to prevent optimization from indirectly modifying the scale-feature via $V_\ell$ [S34]. The featurized scale input is calculated as: $(2 \cdot \operatorname{mean}_j(\downweight_{j, \ell}) - 1)\sqrt{\initval^2 + \stopgrad(\operatorname{mean}(\grid_\ell^2))}$, where $\downweight_{j, \ell}$ is the downweighting factor, $\initval$ is the initialization magnitude of $V_\ell$, and $\grid_\ell$ represents the values in $V_\ell$ [S34]. An approximation for $\operatorname{erf}(x)$ is used for speed: $\operatorname{erf}(x) \approx \operatorname{sign}(x) \sqrt{1 - \exp(-(\sfrac{4}{\pi})x^2)}$ [S34]. The erf approximation has no discernible impact on quality and only a marginal impact on performance [S34].

### Fixed Sine/Cosine Positional Encoding
A baseline variant using fixed sine/cosine positional encoding for 3D Gaussian centers [S35]. On the Brandenburg Gate scene, this method achieves PSNR 18.66, SSIM 0.763, LPIPS 0.347 [S35]. On the Trevi Fountain scene, this method achieves PSNR 19.91, SSIM 0.741, LPIPS 0.305 [S35].

### Gated Positional Encoding (BARF)
A positional encoding scheme where higher frequency bands are multiplied by a time-dependent gating factor $\alpha(t)$ to gradually introduce high-frequency details during training [S36]. The formula is: $\gamma_{\text{BARF}}(x, t) = [x, \sin(2^0 \pi x), \cos(2^0 \pi x), \alpha(t)\sin(2^1 \pi x), \alpha(t)\cos(2^1 \pi x), \dots, \alpha(t)\sin(2^L \pi x), \alpha(t)\cos(2^L \pi x)]$ [S36]. It differs from standard NeRF positional encoding by applying the $\alpha(t)$ gate to frequencies $2^i$ for $i > 0$, while the base frequency $x$ remains ungated [S36]. The number of frequency bands for view-direction encoding, $L_d$, is set to 4 [S36].

## Operating characteristics
- **Time Complexity:** $O(d_{in} \cdot N)$ per query, where $d_{in}$ is input dimension and $N$ is number of frequency bands [S4, S8, S10, S20, unofficial].
- **Memory Complexity:** $O(d_{in} \cdot N)$ for the output tensor [S4, S8, S10, S20, unofficial].
- **Measured Performance:** Requires a larger network (e.g., 8 hidden layers, 256 wide) compared to hash encoding to achieve similar quality, and trains slower [S2, unofficial][S26]. However, it trains over 8× faster than some hash encoding configurations with similar parameter counts due to sparsity of updates and smaller MLP [S2, unofficial].
- **Strengths:** Continuous derivatives allow for second-order derivative computation (e.g., curvature regularization) without gradient locality issues [S13]. Simple to implement and integrate [S1][S6, unofficial][S7].
- **Limitations:** Spectral bias can still affect convergence if frequencies are not well-chosen. Requires careful tuning of `num_freqs` and `max_freq_log2` [S7][S21, unofficial]. High-dimensional P.E. tends to fill the space with high-frequency geometry when the radiance field is underconstrained, resulting in poor generalization [S31, unofficial]. Lightweight architectures struggle to learn from the augmented input signal [S25, unofficial].
- **When to Use:** When continuous derivatives are needed, or when the scene has high-frequency details that ReLU MLPs struggle to learn from raw coordinates [S1, S7, S13].
- **When Not to Use:** When memory or compute is extremely constrained and hash encoding offers sufficient quality with lower overhead [S2, unofficial][S26].

## Background
Fourier Positional Encoding was introduced in the NeRF paper (Mildenhall et al., ECCV 2020) as Eq. 4 [S9, S17, unofficial]. It was inspired by positional encoding in natural language processing (Vaswani et al., 2017) [S2, unofficial][S7]. The technique addresses the spectral bias of MLPs, which prioritize low-frequency signals [S7][S21, unofficial]. It has been widely adopted in neural rendering for encoding spatiodirectionally varying light fields and volume density [S2, unofficial][S26]. Variants include Fourier Feature Networks (Tancik et al., 2020) [S2, unofficial][S7], Integrated Positional Encoding (Barron et al., 2021) [S2, unofficial], and Frequency Annealing (BARF) [S36].

## Sources
[S1] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S2] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
[S3] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S4] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S5] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S6] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S7] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S8] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S9] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S10] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S11] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S12] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S13] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S14] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S15] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27
[S16] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S17] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S18] mdmubarak/OctaNeRF, https://github.com/mdmubarak/OctaNeRF, fetched 2026-10-27 (unofficial)
[S19] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S20] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S21] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
[S22] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S23] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
[S24] Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment, https://doi.org/10.1145/3664647.3680569, fetched 2026-10-27 (unofficial)
[S25] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
[S26] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S27] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S28] NeRF — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-NeRF_SOTA_Updated.html, fetched 2026-10-29
[S29] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S30] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
[S31] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26 (unofficial)
[S32] SWAG: Splatting in the Wild images with Appearance-conditioned Gaussians, https://doi.org/10.48550/arXiv.2403.10427, fetched 2026-10-29 (unofficial)
[S33] MZEN: Multi-zoom Enhanced NeRF for 3-D Reconstruction with Unknown Camera Poses, https://link.springer.com/article/10.1007/s10994-026-07114-x, fetched 2026-10-26 (unofficial)
[S34] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S35] SWAG: Splatting in the Wild images with Appearance-conditioned Gaussians, https://doi.org/10.48550/arXiv.2403.10427, fetched 2026-10-29
[S36] MZEN: Multi-zoom Enhanced NeRF for 3-D Reconstruction with Unknown Camera Poses, https://link.springer.com/article/10.1007/s10994-026-07114-x, fetched 2026-10-26
