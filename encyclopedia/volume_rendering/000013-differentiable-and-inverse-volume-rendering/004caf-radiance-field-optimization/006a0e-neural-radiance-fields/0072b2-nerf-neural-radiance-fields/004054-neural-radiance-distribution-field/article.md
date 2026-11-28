# Neural Radiance Distribution Field

Also called: NeRDF (Neural Radiance Distribution Field)

## Overview
Neural Radiance Distribution Field (NeRDF) is a method for efficient novel view synthesis that combines the 3D prior consistency of NeRF with the single-evaluation rendering speed of NeLF [S1]. Unlike NeRF, which requires multiple network queries per ray, NeRDF maps a camera ray directly to a parametric radiance distribution (opacity and color coefficients) using a small neural network, enabling real-time rendering with low memory cost [S1].

## How it works
NeRDF takes a camera ray $\mathbf{r}$ as input and outputs the coefficients for a trigonometric approximation of the opacity and color distributions along that ray [S1]. These distributions are then integrated via volume rendering to produce the final pixel color [S1].

### Inputs and Outputs
*   **Input:** A camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and distance parameter $t$ [S1].
*   **Output:** A synthesized RGB color $\hat{C}(\mathbf{r})$ for the pixel corresponding to the ray [S1].

### Data Structures and Encoding
The network input is constructed by encoding the ray origin, direction, and sampled points along the ray path [S1].

1.  **Ray Origin ($\mathbf{o}$):** Encoded using frequency-based positional encoding with 10 frequencies [S1].
2.  **Ray Direction ($\mathbf{d}$):** Encoded using spherical harmonic (SH) embedding with 8 degrees, resulting in a 64-dimensional vector [S1].
3.  **Path Points:** $N$ points are sampled along the ray $\mathbf{r}$ using stratified sampling to capture path information [S1]. These $N$ points form a $3N$-dimensional vector, which is embedded using the same frequency-based positional encoding (10 frequencies) as the origin [S1].
4.  **Concatenation:** The encoded origin, encoded direction, and encoded path points are concatenated to form the final input vector to the MLP [S1].

### Network Architecture
The core model is an 8-layer Multi-Layer Perceptron (MLP) with a width of 384 and skip connections between neighboring layers [S1]. The network maps the encoded ray input to two sets of coefficients:
*   $\mathbf{w}^\sigma$: Coefficients for the opacity distribution.
*   $\mathbf{w}^C$: Coefficients for the color distribution.

The total output channels of the MLP is $8K$, where $K$ is the number of frequencies in the Fourier approximation [S1].

### Parametric Radiance Distribution
The opacity $\sigma(t)$ and color $C(t)$ along the ray distance $t$ are parameterized using a Fourier series with $2K$ coefficients each and trigonometric basis functions $\mathcal{T}_i(t)$ [S1]:

$$
\sigma(t) = \sum_{i=0}^{2K-1} w_i^\sigma \cdot \mathcal{T}_i(t)
$$

$$
C(t) = \sum_{i=0}^{2K-1} w_i^C \cdot \mathcal{T}_i(t)
$$

The basis functions $\mathcal{T}_i(t)$ are defined based on the ray length $T$ and index $i$ [S1]:

$$
\mathcal{T}_i(t) = 
\begin{cases} 
\cos\left(\frac{i\pi}{T}t\right) & \text{for even } i \\
\sin\left(\frac{(i+1)\pi}{T}t\right) & \text{for odd } i 
\end{cases}
$$

### Rendering Process
The final pixel color is computed via volume rendering integration using the predicted distributions [S1].

1.  **Sampling:** Sample $M$ points along the ray $\mathbf{r}$ to obtain distance stamps $t_m$ (typically $M=64$ uniform samples) [S1].
2.  **Evaluation:** Evaluate $\sigma(t_m)$ and $C(t_m)$ at each sampled point using the Fourier series formulas above [S1].
3.  **Integration:** Compute the final pixel color $\hat{C}(\mathbf{r})$ using standard volume rendering integration [G]:

$$
\hat{C}(\mathbf{r}) = \sum_{m=1}^{M} T_m \left(1 - e^{-\sigma(t_m)\delta_m}\right) C(t_m)
$$

where $\delta_m$ is the distance between adjacent samples and $T_m = e^{-\sum_{j=1}^{m-1} \sigma(t_j)\delta_j}$ is the accumulated transmittance [G].

### Training Strategy
NeRDF is trained using a knowledge distillation framework where a teacher NeRF synthesizes pseudo-data [S1].

*   **Online View Sampling (OVS):** Instead of pre-computing pseudo-data, camera poses are randomly sampled during training. A batch consists of 1 view origin and 2,048 directions [S1].
*   **Loss Function:** The primary loss is the L2 norm between the synthesized color and the ground truth (or teacher pseudo-data) color [S1]:

$$
L_{render} = \sum_{\mathbf{r}} \|\hat{C}(\mathbf{r}) - C(\mathbf{r})\|_2^2
$$

*   **Volume Density Constraint (VDC):** To ensure multi-view consistency and valid 3D priors, a VDC loss minimizes the L2 distance between the normalized volume density predicted by NeRDF ($\hat{\sigma}_1(t)$) and the teacher NeRF ($\hat{\sigma}_2(\mathbf{r}(t))$) [S1]:

$$
M_{vdc} = \lambda \sum_{\mathbf{r}} \sum_t \|\hat{\sigma}_1(t) - \hat{\sigma}_2(\mathbf{r}(t))\|_2^2
$$

where $\lambda$ is a weighting factor (typically 0.1) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def positional_encoding(x, num_frequencies):
    """Frequency-based positional encoding."""
    encoded = []
    for val in x:
        for i in range(num_frequencies):
            freq = 2.0 ** i
            encoded.append(math.sin(val * freq))
            encoded.append(math.cos(val * freq))
    return encoded

def spherical_harmonic_encoding(direction, degrees):
    """Spherical harmonic encoding for direction.
    [G] Standard SH basis evaluation for 8 degrees yields 64 dimensions."""
    return [0.0] * 64

def evaluate_basis(t, T, K):
    """Evaluate trigonometric basis functions T_i(t) for i in 0..2K-1."""
    basis = []
    for i in range(2 * K):
        if i % 2 == 0:
            basis.append(math.cos((i * math.pi / T) * t))
        else:
            basis.append(math.sin(((i + 1) * math.pi / T) * t))
    return basis

def nerdf_forward(origin, direction, t_samples, K, width, num_layers):
    # 1. Encode origin (3,)
    o_enc = positional_encoding(origin, 10)
    
    # 2. Encode direction (3,)
    d_enc = spherical_harmonic_encoding(direction, 8)
    
    # 3. Sample points along ray: p = o + t * d
    points = [[origin[j] + t * direction[j] for j in range(3)] for t in t_samples]
    
    # 4. Encode points
    p_enc = []
    for p in points:
        p_enc.extend(positional_encoding(p, 10))
        
    # 5. Concatenate input
    x = o_enc + d_enc + p_enc
    
    # 6. MLP forward with skip connections
    h = x
    for i in range(num_layers):
        # Linear transformation (simulated)
        h = [sum(h[j] * 0.1 for j in range(len(h))) for _ in range(width)]
        if i < num_layers - 1:
            h = [max(0.0, v) for v in h]
        if i > 0:
            h = [h[j] + x[j] for j in range(len(h))]
        x = h
        
    # Output coefficients
    coeffs = [sum(x[j] * 0.1 for j in range(len(x))) for _ in range(8 * K)]
    w_sigma = coeffs[:4 * K]
    w_C = coeffs[4 * K:]
    
    # Evaluate distributions
    T = t_samples[-1] - t_samples[0] if len(t_samples) > 1 else 1.0
    sigma_dist = []
    C_dist = []
    for t in t_samples:
        basis = evaluate_basis(t, T, K)
        sigma_val = sum(w_sigma[i] * basis[i] for i in range(2 * K))
        C_val = sum(w_C[i] * basis[i] for i in range(2 * K))
        sigma_dist.append(sigma_val)
        C_dist.append(C_val)
        
    return sigma_dist, C_dist
```

### Parameters
*   **K (Frequencies):** 12. Robust range 4–24 [S1].
*   **MLP Width:** 384 [S1].
*   **MLP Layers:** 8 [S1].
*   **N (Path Points):** 16 [S1].
*   **SH Degrees:** 8 (64-dim direction encoding) [S1].
*   **Positional Encoding Frequencies:** 10 [S1].
*   **Rendering Samples:** 64 uniform samples [S1].
*   **Batch Size:** 1 view origin, 2,048 directions [S1].
*   **Learning Rate:** $5 \times 10^{-4}$ [S1].
*   **Iterations:** 400k–600k [S1].
*   **VDC Weight ($\lambda$):** 0.1 [S1].

### Edge Cases and Pitfalls
*   **Ray Length ($T$):** The basis functions depend on $T$ [S1].
*   **Direction Encoding:** Omitting direction encoding drops performance by ~0.1 dB [S1].
*   **Path Points:** Omitting path points drops performance by 1.1 dB [S1].
*   **Network Size:** Using too small a network (e.g., R2L-16 equivalent) may result in inadequate quality, though NeRDF-8 is robust [S1].

## Operating characteristics
*   **Speed:** NeRDF-8 achieves ~21 FPS on RTX3090 without optimization, and ~369 FPS with `tiny-cuda-nn` optimization [S1]. This is a ~254× speed-up over NeRF and ~6-10× over R2L [S1].
*   **Memory:** NeRDF-8 uses 5.1 MB memory, significantly less than R2L-88 (23.0 MB) and NeX (89.0 MB) [S1].
*   **Quality:** NeRDF-8 achieves 26.53 dB PSNR on LLFF, comparable to R2L-88 (27.79 dB) but with much higher speed [S1].
*   **Strengths:** Real-time rendering, low memory, small network size, 3D prior consistency [S1].
*   **Limitations:** Not designed for 360-degree scenes with high fidelity [S1].

## Background
NeRDF was proposed by Wu et al. in 2023 [S1]. It builds on the knowledge distillation framework of R2L [S1] and aims to bridge the gap between NeRF's quality and NeLF's speed [S1].

## Sources
[S1] Efficient View Synthesis with Neural Radiance Distribution Field, https://doi.org/10.1109/iccv51070.2023.01696, fetched 2026-10-26
