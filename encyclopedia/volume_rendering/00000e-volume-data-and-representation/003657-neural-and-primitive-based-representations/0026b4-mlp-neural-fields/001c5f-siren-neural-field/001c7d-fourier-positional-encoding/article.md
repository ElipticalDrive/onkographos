# Fourier Positional Encoding

Also called: Fourier Features, Sinusoidal Encoding, Sinusoidal Embedding, Fourier Feature Mappings, Fourier Feature Embedding, Sinusoidal Positional Encoding, Fourier Feature Positional Encoding

## Overview
Fourier Positional Encoding maps low-dimensional input coordinates (such as 3D positions or viewing directions) into a higher-dimensional space using a multi-resolution sequence of sine and cosine functions at exponentially increasing frequencies [S1][S2, S3, unofficial][S7]. It solves the problem of spectral bias in neural networks, where standard architectures (particularly those with ReLU activations) struggle to learn high-frequency details from raw, low-frequency inputs [S7][S20, unofficial]. By lifting inputs to a Fourier feature space, the method enables the subsequent MLP to represent fine-scale scene variations more effectively [S1, S7]. It differs from its parent [SIREN Neural Field](1250-siren-neural-field.md) by serving as a fixed preprocessing step for the MLP input rather than relying on sine activations within the network layers themselves, and from [Uncertainty-Aware SIREN](10449-uncertainty-aware-siren.md) by not explicitly modeling uncertainty in the encoding domain [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of coordinates $x \in \mathbb{R}^{d_{in}}$ (e.g., $d_{in}=3$ for 3D position, or $d_{in}=2$ for viewing direction angles).
*   **Output:** A tensor of encoded features $z \in \mathbb{R}^{d_{out}}$.

**Data Structures and Parameters:**
*   `input_dims` ($d_{in}$): Dimensionality of the input vector.
*   `num_freqs` ($N$ or $L$): Number of frequency bands (octaves). Typical value: 10 for position, 4 for direction [S9, S12, S25, unofficial].
*   `log_sampling` (bool): Whether frequencies are spaced logarithmically (powers of 2) or linearly. Default: True [S4, S5, S6, unofficial].
*   `include_input` (bool): Whether to concatenate the original raw input $x$ to the encoded output. Default: True [S5, S6, S15, unofficial].
*   `omega0` (float): Frequency scaling factor. Default: 1.0 [S4, S8, unofficial].
*   `max_freq_log2` (float): The maximum frequency exponent. If `log_sampling` is True, the highest frequency is $2^{\text{max\_freq\_log2}}$. Often set to `num_freqs - 1` [S6, S19, S21, unofficial].

**Formulas:**
The encoding function $\gamma(x)$ for a scalar input $x$ is defined as a concatenation of sine and cosine terms at different frequencies $f_k$:
$$ \gamma(x) = [\sin(2\pi f_1 x), \cos(2\pi f_1 x), \dots, \sin(2\pi f_N x), \cos(2\pi f_N x)] $$
If `include_input` is True, the raw input is prepended:
$$ \gamma_{\text{full}}(x) = [x, \sin(2\pi f_1 x), \cos(2\pi f_1 x), \dots, \sin(2\pi f_N x), \cos(2\pi f_N x)] $$

The output dimension $d_{out}$ is calculated as:
$$ d_{out} = d_{in} \times (2 \times N + \mathbb{I}_{\text{include\_input}}) $$
where $\mathbb{I}$ is the indicator function (1 if true, 0 if false) [S4, S8, S19, unofficial].

**Frequency Generation:**
1.  If `log_sampling` is True:
    $$ f_k = 2^k \quad \text{for } k \in [0, N-1] $$
    Alternatively, if `max_freq_log2` is specified:
    $$ f_k = 2^{k \cdot \frac{\text{max\_freq\_log2}}{N-1}} \quad \text{for } k \in [0, N-1] $$
    [S4, S5, S6, S15, S21, unofficial]
2.  If `log_sampling` is False (linear spacing):
    $$ f_k = \text{linspace}(1.0, 2^{\text{max\_freq\_log2}}, N) $$
    [S5, S6, S15, S21, unofficial]

**Numbered Steps:**
1.  Generate the frequency bands $F = [f_1, \dots, f_N]$ based on `log_sampling` and `max_freq_log2` [S4, S5, S6, unofficial].
2.  Scale the frequencies by `omega0`: $f_k \leftarrow f_k \cdot \omega_0$ [S4, S8, unofficial].
3.  If `include_input` is True, initialize the output list with the original input $x$ [S5, S6, S15, unofficial].
4.  For each frequency $f_k$ in $F$:
    a. Compute $s_k = \sin(2\pi f_k x)$ and $c_k = \cos(2\pi f_k x)$ [S1][S9, S15, unofficial].
    b. Append $s_k$ and $c_k$ to the output list [S15, S18, unofficial].
    *Note: Some implementations use the identity $\cos(x) = \sin(x + \pi/2)$ to compute both with a single vectorized sine call [S22, unofficial].*
5.  Concatenate all elements in the output list along the last dimension to produce the final embedding $z$ [S15, S16, S18, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def fourier_pos_encoding(x, num_freqs=10, log_sampling=True, include_input=True, omega0=1.0, max_freq_log2=None):
    """
    Computes Fourier Positional Encoding.
    
    Args:
        x: Input tensor of shape (*, d_in).
        num_freqs: Number of frequency bands (N).
        log_sampling: If True, frequencies are powers of 2. If False, linearly spaced.
        include_input: If True, concatenate raw x to output.
        omega0: Frequency scaling factor.
        max_freq_log2: Maximum frequency exponent. If None, defaults to num_freqs - 1.
        
    Returns:
        Encoded tensor of shape (*, d_out).
    """
    if max_freq_log2 is None:
        max_freq_log2 = num_freqs - 1
        
    # Step 1 & 2: Generate and scale frequencies
    if log_sampling:
        # Powers of 2 from 2^0 to 2^max_freq_log2
        # Using linspace to handle the case where num_freqs != max_freq_log2 + 1
        freq_bands = 2.0 ** torch.linspace(0.0, max_freq_log2, num_freqs, device=x.device, dtype=x.dtype)
    else:
        # Linear spacing from 1.0 to 2^max_freq_log2
        freq_bands = torch.linspace(1.0, 2.0 ** max_freq_log2, num_freqs, device=x.device, dtype=x.dtype)
        
    freq_bands = freq_bands * omega0
    
    # Step 3: Prepare output parts
    parts = []
    if include_input:
        parts.append(x)
        
    # Step 4: Compute sine and cosine for each frequency
    # Reshape x to (*, 1, d_in) and freq_bands to (num_freqs, 1, 1) for broadcasting
    # x shape: (*, d_in)
    # freq_bands shape: (num_freqs,)
    
    # Expand dimensions for broadcasting: x -> (*, 1, d_in), freq -> (num_freqs, 1, 1)
    # However, standard broadcasting works if we align the last dim of x with the freq dim
    # x[..., None] -> (*, d_in, 1)
    # freq_bands[None, None, :] -> (1, 1, num_freqs)
    # Result: (*, d_in, num_freqs)
    
    scaled_inputs = x[..., None] * freq_bands[None, None, :] # Shape: (*, d_in, num_freqs)
    
    # Compute sin and cos
    sin_vals = torch.sin(2 * math.pi * scaled_inputs)
    cos_vals = torch.cos(2 * math.pi * scaled_inputs)
    
    # Interleave sin and cos for each frequency and dimension
    # We want output shape: (*, d_in * 2 * num_freqs)
    # Current shape: (*, d_in, num_freqs)
    
    # Flatten the last two dimensions to interleave
    # sin_vals: (*, d_in, num_freqs) -> (*, d_in * num_freqs)
    # cos_vals: (*, d_in, num_freqs) -> (*, d_in * num_freqs)
    
    sin_flat = sin_vals.reshape(*x.shape[:-1], -1)
    cos_flat = cos_vals.reshape(*x.shape[:-1], -1)
    
    # Concatenate sin and cos
    encoded = torch.cat([sin_flat, cos_flat], dim=-1)
    
    if include_input:
        parts.append(encoded)
        return torch.cat(parts, dim=-1)
    else:
        return encoded
```

**Edge Cases and Pitfalls:**
*   **Lambda Closure Issue:** When implementing this as a list of lambda functions (e.g., in TensorFlow or older PyTorch patterns), variables `p_fn` and `freq` must be bound explicitly as default arguments (`lambda x, p_fn=p_fn, freq=freq: p_fn(x * freq)`) to avoid capturing the final loop values [S19, unofficial].
*   **Precision:** Directly regressing shape properties from raw coordinates fails due to spectral bias; the encoding is necessary to overcome this [S7].
*   **Derivatives:** The derivative of the encoding is continuous across space, allowing for second-order derivative computations (e.g., curvature regularization) without the gradient locality issues seen in hash encodings [S13].

## Subtopics

### [Fourier Feature Networks](2765-fourier-feature-networks.md)
Replaces the learning burden of the featurizer with a fixed, high-dimensional Fourier embedding to remedy spectral bias in ReLU MLPs.

### [Off-Axis Positional Encoding](3503-off-axis-positional-encoding.md)
Uses a non-identity basis matrix (icosahedron-derived) to capture off-axis correlations for anisotropic Gaussians, solving ambiguity in axis-aligned encodings.

### [Coordinate-Time Encoding](5105-coordinate-time-encoding.md)
A continuous, single-scale variant (One-Blob Encoding) for bounded domains, providing localized representation at the cost of multi-scale support.

### [Integrated Positional Encoding](5255-integrated-positional-encoding.md)
Integrates sine/cosine features over an interval (cone) to anti-alias high-frequency features, solving aliasing artifacts from multi-resolution observation.

### [Integrated Positional Encoding (Diagonal Covariance)](12189-integrated-positional-encoding-diagonal-covariance.md)
Computes the expected value of sinusoidal encodings for Gaussian random variables with diagonal covariance, acting as a low-pass filter.

### [2D Positional Embedding](12829-2d-positional-embedding.md)
Explicitly handles 2D coordinate tensors $(x, y)$ and often concatenates original coordinates, distinguishing itself from siblings that treat frequency matrices as trainable.

### [Frequency Annealing](13443-frequency-annealing.md)
A training regularization strategy that gradually activates higher-frequency bands to prevent optimization instability and local minima.

### [Truncated Dynamic Low-pass Filter](14016-truncated-dynamic-low-pass-filter.md)
A frequency masking strategy that initializes the frequency threshold at a non-zero value to ensure valid color information while suppressing high-frequency details for robust pose estimation.

### Scale Featurization
A method to provide scale information to the MLP by averaging and concatenating a featurized version of downweighting factors $\omega_{j,\ell}$, ensuring features have consistent magnitude regardless of training progression [S28]. It scales the result by the standard deviation of values in $V_\ell$, padded with $\initval$ to prevent zero values, and applies a stop-gradient operator to the mean of the squared grid values to prevent optimization from indirectly modifying the scale-feature via $V_\ell$ [S28]. The featurized scale input is calculated as: $(2 \cdot \operatorname{mean}_j(\downweight_{j, \ell}) - 1)\sqrt{\initval^2 + \stopgrad(\operatorname{mean}(\grid_\ell^2))}$, where $\downweight_{j, \ell}$ is the downweighting factor, $\initval$ is the initialization magnitude of $V_\ell$, and $\grid_\ell$ represents the values in $V_\ell$ [S28]. An approximation for $\operatorname{erf}(x)$ is used for speed: $\operatorname{erf}(x) \approx \operatorname{sign}(x) \sqrt{1 - \exp(-(\sfrac{4}{\pi})x^2)}$, which has no discernible impact on quality and only a marginal impact on performance [S28].

### Conditional Positional Encoding
A positional encoding technique that applies phase shifts based on time index to sampled features, inspired by Fourier transformation phase information retention [S29]. It applies positional encoding to 6D attributes (x, y, z, t, θ, ϕ) and sampled features (Fden, Ftex), explicitly assigning phase shift to different frequencies using time index t [S29]. The frequency order L is typically 5 [S29]. CPE is not applied to density field features to maintain zero initialization for correct volume rendering [S29].

## Operating characteristics
*   **Time Complexity:** The encoding computation is $O(d_{in} \cdot N)$ per sample, where $N$ is the number of frequency bands. This is negligible compared to the MLP forward pass.
*   **Memory Complexity:** The output dimension is $d_{out} = d_{in} \times (2N + \mathbb{I}_{\text{include\_input}})$. For $d_{in}=3, N=10$, this results in 63 features per coordinate (189 total for x,y,z) [S12, unofficial]. This increases the input size to the MLP, requiring more parameters to maintain capacity.
*   **Measured Performance:** In NeRF, using Fourier encoding with $L=10$ for position and $L=4$ for direction is standard [S12, S25, unofficial]. It requires a larger network (e.g., 8 hidden layers, 256 wide) compared to hash encoding to achieve similar quality and trains slower [S2, unofficial][S23]. However, it is more parameter-efficient than some alternatives and avoids the gradient locality issues of hash encodings [S13].
*   **Strengths:** Continuous derivatives allow for higher-order derivative computations (curvature regularization) [S13]. Simple to implement and widely supported.
*   **Limitations:** Susceptible to aliasing when rendering views at varying resolutions or disambiguating anisotropic structures [S7]. Requires careful frequency selection (annealing) to avoid optimization instability [S13443-frequency-annealing.md].
*   **When to Use:** When the scene contains high-frequency details that ReLU MLPs struggle to learn, and when derivative continuity is required.
*   **When Not to Use:** When memory is constrained (due to high output dimensionality) or when aliasing is a major issue without integrated encoding [S5255-integrated-positional-encoding.md].

## Background
Fourier Positional Encoding was introduced in the original NeRF paper (Mildenhall et al., ECCV 2020) [S9, unofficial][S1]. It was inspired by positional encoding techniques used in natural language processing (Vaswani et al., 2017) [S2, unofficial][S7]. The method was adopted to encode the five dimensions of the spatiodirectionally varying light field and volume density independently [S2, unofficial][S23]. It addresses the spectral bias of MLPs, which inherently prioritize smooth, low-frequency signals [S7][S20, unofficial]. The technique has since been extended to randomly oriented parallel wavefronts [S2, unofficial] and level-of-detail filtering [S2, unofficial]. It is a core component of the SIREN Neural Field [1250-siren-neural-field.md] and its variants.

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
[S15] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S16] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S17] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S18] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S19] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S20] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
[S21] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S22] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
[S23] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S24] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S25] NeRF — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-NeRF_SOTA_Updated.html, fetched 2026-10-29 (unofficial)
[S26] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S27] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
[S28] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S29] V4D: Voxel for 4D Novel View Synthesis, https://doi.org/10.1109/tvcg.2023.3312127, fetched 2026-10-26
