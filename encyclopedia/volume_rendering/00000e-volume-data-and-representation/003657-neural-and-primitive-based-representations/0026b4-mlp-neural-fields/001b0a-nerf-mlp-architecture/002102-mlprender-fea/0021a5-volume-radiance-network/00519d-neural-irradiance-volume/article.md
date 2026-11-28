# Neural Irradiance Volume

## Overview
Neural Irradiance Volume (NIV) is a real-time rendering technique that compresses the 5D indirect diffuse irradiance field of a static scene into a compact neural model, enabling the shading of unseen dynamic objects without runtime ray tracing or denoising [S1]. Unlike traditional grid-based probe methods which suffer from aliasing and cubic memory scaling, NIV maps position and direction directly to irradiance values, improving quality by at least 10x at the same memory budget [S1]. It differs from other neural rendering methods by learning a pre-integrated volumetric quantity (irradiance) rather than raw incident radiance, allowing noise-free shading at ~1 ms per frame on consumer hardware [S1].

## How it works
**Inputs and Outputs**
*   **Training Input:** Position-direction pairs $(x, n)$ and ground-truth indirect irradiance $E(x, n)$ computed via path tracing [S1].
*   **Runtime Input:** Position buffer $\mathbf{x}$ and Normal buffer $\mathbf{n}$ from the scene's G-buffer [S1].
*   **Runtime Output:** Indirect diffuse irradiance $E(x, n)$ per pixel, used to compute reflected radiance [S1].

**Data Structures**
The core data structure is a neural network $E_\theta$ combined with an input encoding scheme [S1]. Two variants are described:
1.  **Small Models:** Use frequency encoding (positional encoding) for input position $x$ [S1].
2.  **Large Models:** Use a learned multi-level hash-grid encoding [MESK22] [S1].
    *   **Hash Table:** Stores latent vectors for finer levels [S1]. Size $T = 2^{17}$ is used for all experiments [S1].
    *   **Encoding Parameters:** Latent dimension size is 4; scaling factor between levels is 2; side length of the coarsest level is 16; number of levels varies between 2 and 8 [S1].
    *   **Network Architecture:** A four-layer fully connected neural network with ReLU activations at all but the final layer [S1].

**Formulas**
The method factors out the BRDF to learn only the incoming irradiance $E$ [S1].
1.  **Indirect Irradiance Definition:**
    $$ E(x, n) = \int_{\Omega} L_i(x, \omega_i) \langle \omega_i, n \rangle_+ d\omega_i $$
    Where $E$ is indirect irradiance, $x$ is position, $n$ is the normal/direction, $L_i$ is incoming radiance, $\Omega$ is the spherical domain, and $\langle \cdot, + \rangle$ denotes the positive dot product [S1].

2.  **Diffuse Reflected Radiance:**
    $$ L_r(x) = \frac{\rho(x)}{\pi} \cdot E(x, n) $$
    Where $L_r$ is diffuse reflected radiance, $\rho(x)$ is surface albedo, and $E(x, n)$ is the queried indirect irradiance [S1].

3.  **Training Loss (Relative L2):**
    $$ \mathcal{L}_\theta(E(x, n), E_\theta(x, n)) = \frac{(E_\theta(x, n) - E(x, n))^2}{\text{sg}(E_\theta(x, n)^2) + \epsilon} $$
    Where $\text{sg}$ is the stop-gradient operation and $\epsilon = 0.01$ [S1].

**Parameters**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $10^{-2}$, decaying exponentially to $10^{-4}$ after 10,000 iterations [S1].
*   **Batch Size:** $2^{16}$ [S1].
*   **Convergence:** At most 50,000 iterations [S1].
*   **Training Sample Distribution:** 20% of samples are allocated to static surfaces with direction $n$ set to the surface normal; the remaining 80% are uniformly sampled in the volume [S1].
*   **Inference Resolution:** Can be run at half-resolution (quarter pixel count) to reduce time, using full-resolution albedo for material accuracy [S1].

**Numbered Steps**

1.  **Precomputation (Training):**
    1.  Sample position-direction pairs $(x, n)$ throughout the scene volume [S1]. Allocate 20% of samples to static surface geometry, setting $n$ to the surface normal [S1].
    2.  For each sample, compute ground-truth indirect irradiance $E(x, n)$ via path tracing [S1].
    3.  Discard samples located inside scene geometry [S1]. Identify these by checking if the majority of normals at the first hit are backfacing [S1].
    4.  Encode input position $x$ using either frequency encoding (small models) or multi-level hash-grid encoding (large models) [S1].
    5.  Compute the network output $E_\theta(x, n)$ and the relative L2 loss $\mathcal{L}_\theta$ [S1].
    6.  Update network parameters $\theta$ using the Adam optimizer with the specified learning rate schedule [S1].
    7.  Repeat until convergence (max 50k iterations) [S1].

2.  **Runtime Inference:**
    1.  Render the static scene to generate a G-buffer containing position $\mathbf{x}$ and normal $\mathbf{n}$ buffers [S1].
    2.  (Optional) Perform a dynamic ambient occlusion pass considering only dynamic geometry to compute hemisphere obstruction for self-occlusion and indirect occlusion on static surfaces [S1].
    3.  Input the position and normal buffers into the trained neural model $E_\theta$ to infer pixel-wise $E(x, n)$ in a single batch [S1].
    4.  Compute indirect diffuse radiance using $L_r(x) = \frac{\rho(x)}{\pi} \cdot E(x, n)$, where $\rho(x)$ is the full-resolution albedo buffer [S1].
    5.  Add direct illumination and surface emission to obtain the final frame [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class NeuralIrradianceVolume:
    def __init__(self, use_hash_encoding=True, hash_table_size=2**17, latent_dim=4, num_levels=8):
        if use_hash_encoding:
            # Hash grid encoding implementation is from [MESK22], not detailed in sources.
            # Parameters: latent_dim=4, scale_factor=2, coarse_side=16.
            self.encoder = HashGridEncoderStub(num_levels=num_levels, latent_dim=latent_dim)
        else:
            # Frequency encoding for smaller models [MST*21]
            self.encoder = FrequencyEncoder()
        
        # Four-layer fully connected network with ReLU activations
        self.network = FullyConnectedNetwork(self.encoder.output_dim + 3)
        self.optimizer = AdamOptimizer(self.network.parameters(), lr=1e-2)
        self.epsilon = 0.01

    def forward(self, positions, normals):
        encoded_pos = self.encoder(positions)
        input_tensor = np.concatenate([encoded_pos, normals], axis=-1)
        irradiance = self.network(input_tensor)
        return irradiance

    def compute_loss(self, predicted_irradiance, ground_truth_irradiance):
        diff = predicted_irradiance - ground_truth_irradiance
        numerator = diff ** 2
        # sg() denotes stop gradient operation [S1]
        denominator = np.square(predicted_irradiance) + self.epsilon
        loss = np.mean(numerator / denominator)
        return loss

    def train_step(self, positions, normals, ground_truth_irradiance):
        predicted = self.forward(positions, normals)
        loss = self.compute_loss(predicted, ground_truth_irradiance)
        self.optimizer.step(loss)
        return loss

    def render_frame(self, g_buffer_position, g_buffer_normal, g_buffer_albedo):
        pos_flat = g_buffer_position.reshape(-1, 3)
        norm_flat = g_buffer_normal.reshape(-1, 3)
        irradiance = self.forward(pos_flat, norm_flat)
        irradiance_img = irradiance.reshape(g_buffer_position.shape)
        pi = np.array([math.pi])
        radiance = (g_buffer_albedo / pi) * irradiance_img
        # Add direct illumination and emission as per [S1]
        return radiance

class FrequencyEncoder:
    def __init__(self):
        self.output_dim = 16 # 8-band positional encoding [MST*21]

    def forward(self, x):
        # Standard positional encoding [G]
        scales = np.array([2**i for i in range(self.output_dim // 2)])
        x = x[..., np.newaxis] * scales
        x = np.concatenate([np.sin(x), np.cos(x)], axis=-1)
        return x.reshape(x.shape[0], -1)

class HashGridEncoderStub:
    def __init__(self, num_levels, latent_dim):
        self.num_levels = num_levels
        self.latent_dim = latent_dim
        self.output_dim = latent_dim * num_levels
        # Implementation details abstracted per [MESK22]

    def forward(self, x):
        # Placeholder for hash grid lookup and interpolation [G]
        return np.zeros(x.shape[:-1] + (self.output_dim,))

class FullyConnectedNetwork:
    def __init__(self, input_dim):
        self.weights = [np.random.randn(input_dim, 64), np.random.randn(64, 64), np.random.randn(64, 64), np.random.randn(64, 3)]
        self.biases = [np.zeros(64), np.zeros(64), np.zeros(64), np.zeros(3)]

    def forward(self, x):
        for w, b in zip(self.weights, self.biases):
            x = x @ w + b
            if w is not self.weights[-1]:
                x = np.maximum(x, 0) # ReLU
        return x

class AdamOptimizer:
    def __init__(self, params, lr):
        self.lr = lr
        self.params = params
        self.m = [np.zeros_like(p) for p in params]
        self.v = [np.zeros_like(p) for p in params]
        self.t = 0

    def step(self, loss):
        self.t += 1
        # Simplified Adam update step [G]
        for i, p in enumerate(self.params):
            grad = np.gradient(loss, p) # Placeholder for actual gradient computation
            self.m[i] = 0.9 * self.m[i] + 0.1 * grad
            self.v[i] = 0.999 * self.v[i] + 0.001 * grad**2
            self.params[i] -= self.lr * self.m[i] / (np.sqrt(self.v[i]) + 1e-8)
```

**Edge Cases and Pitfalls**
*   **Hash Collisions:** Many positions may map to the same latent vectors in the hash table. These are implicitly handled during optimization as gradients from more important samples dominate the averaged gradient. However, too many collisions reduce reconstruction quality [S1].
*   **Dark Leaks:** Samples inside geometry can waste capacity and cause dark leaks near static surfaces. This is mitigated by discarding samples where the majority of normals at the first hit are backfacing [S1].
*   **Aliasing:** Running at half-resolution introduces aliasing artifacts. However, indirect irradiance varies smoothly in screen-space, making artifacts minimal, especially when multiplied by the full-resolution albedo buffer [S1].

## Subtopics

### Disco NSVF Models
Neural Scene Volume Fields that use discrete coordinates and sparse volumes [G]. Unlike NIV, which learns a pre-integrated irradiance field for diffuse shading, NSVF models typically represent radiance or density directly [S1]. Choose NSVF for general neural radiance field reconstruction; choose NIV for real-time diffuse global illumination of dynamic objects in static scenes [S1, G].

## Operating characteristics
*   **Time Complexity:** Inference time is approximately 1 ms per frame on consumer GPUs (RTX 4090) at full HD resolution for the largest model; 0.37 ms at half-resolution [S1]. Smallest models take 0.19 ms (full) or 0.029 ms (half) [S1].
*   **Memory Complexity:** Memory usage is 1–5 MB for medium-sized scenes, significantly lower than grid-based methods which scale cubically [S1].
*   **Performance:** On RTX 4090 and i9-13900K, full HD inference for the largest model takes 1.35 ms (full) or 0.37 ms (half) [S1].
*   **Strengths:** Heuristic-free optimization via backpropagation; avoids cubic scaling of grid-based methods; enables noise-free shading of dynamic objects without ray tracing; high quality improvement (10x) at same memory budget compared to probe methods [S1].
*   **Limitations:** Requires precomputation (training) time (minutes to hours depending on scene size); relies on static scene geometry for the irradiance field (dynamic objects are shaded against the precomputed field); hash collisions can reduce quality if table size is too small [S1].
*   **When to Use:** Real-time rendering of diffuse global illumination in scenes with static environments and dynamic objects, where memory and latency are constrained [S1].
*   **When Not to Use:** Scenes with dynamic lighting or dynamic geometry that significantly alters the irradiance field (NIV assumes static irradiance field); scenes requiring specular or non-diffuse global illumination (NIV factors out BRDF for diffuse only) [S1].

## Background
Neural Irradiance Volume was proposed by Coomans and Nazzaro et al. in the paper "Real-time Rendering with a Neural Irradiance Volume" [S1]. The method addresses the limitations of traditional irradiance probe methods, which suffer from aliasing and high memory consumption due to cubic scaling [S1]. It builds on the concept of neural fields and hash encodings [MESK22] to provide a compact, adaptive representation of the 5D irradiance field [S1].

## Sources
[S1] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
