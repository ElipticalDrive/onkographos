# Neural Volumes with Multi-Resolution Hash Encoding

Also called: Hash Encoding Neural Volume, InstantVNR, Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation, VNR

## Overview
Neural Volumes with Multi-Resolution Hash Encoding is a method for interactive volume rendering that compresses volumetric data into a neural representation using multi-resolution hash grids, enabling real-time ray tracing on modern GPUs [S1, unofficial]. It distinguishes itself from other neural field methods by supporting three distinct decoding paths—dense 3D texture, sample streaming, and in-shader inference—and by integrating macro-cell acceleration for adaptive empty-space skipping, achieving 10-60fps rendering with high fidelity (PSNR > 30dB) and compact storage (10-1000x smaller than raw data) [S1, unofficial]. Unlike [NeuS2 Network Architecture](10215-neus2-network-architecture.md) which focuses on surface reconstruction, this method targets volumetric density and color, and unlike [HashGrid Encoding](3866-hashgrid-encoding.md) which is a general feature extractor, this method provides a complete rendering pipeline optimized for volume data.

## How it works
The method defines a scene function $F_\Theta: \mathbb{R}^3 \rightarrow \mathbb{R}^k$ where the input is a 3D spatial position $\mathbf{x}$ and the output is a signal vector containing density $\sigma$ and color $\mathbf{c}$ [G]. The representation is trained by minimizing the loss between rendered observations and ground truth images [G]. The core innovation is the use of multi-resolution hash encoding to store spatial features, which are interpolated and concatenated before being passed to a decoder network [G].

### Inputs and Outputs
*   **Input:** 3D spatial coordinates $\mathbf{x}$, viewing direction $\mathbf{d}$ (optional, for view-dependent effects), and transfer function parameters.
*   **Output:** Rendered image pixels, each composed of color $\mathbf{C}$ and opacity $\alpha$.

### Data Structures
The implementation relies on specific CUDA device-side structures for ray marching and payload management.

**Ray Structure**
*   `vec3f org`: Ray origin.
*   `vec3f dir`: Ray direction.
*   `float alpha`: Current accumulated opacity (default 0.f).
*   `vec3f color`: Current accumulated color (default 0.f, not used by shadow rays).

**SampleStreamingPayload Struct**
Used for collecting coordinates for host-side inference.
*   `uint32_t pixel_index`: Index of the pixel being processed.
*   `float jitter`: Jitter value for sampling.
*   `RayMarchingIter iter`: Iterator state for the current ray segment.
*   `union { vec3f color; vec3f org; }`: Union storing either intermediate color or origin.
*   `float alpha`: Intermediate opacity.

**SingleShotPayload Struct**
Used for single-shot heuristic shading.
*   `vec3f highest_org`: Highest density origin.
*   `vec3f highest_color`: Highest density color.
*   `float highest_alpha`: Highest density opacity.

**Buffer Allocation (Iterative Mode)**
For adaptive sampling modes, a contiguous buffer is allocated with the following layout (pointers assigned in order):
1.  `inference_input`: Size `numPixelsPadded * nSamplesPerCoord * sizeof(vec4f)`.
2.  `inference_output`: Size `numPixelsPadded * nSamplesPerCoord * sizeof(vec4f)`.
3.  `alpha`: Size `numPixelsPadded * sizeof(float)`.
4.  `color_or_org`: Size `numPixelsPadded * sizeof(vec3f)`.
5.  `pixel_index`: Size `numPixelsPadded * sizeof(uint32_t)`.
6.  `jitter`: Size `numPixelsPadded * sizeof(float)`.
7.  `iter_cell` (if adaptive): Size `numPixelsPadded * sizeof(uint32_t)`.
8.  `iter_t_next` (if adaptive): Size `numPixelsPadded * sizeof(float)`.
9.  `iter_next_cell_begin` (if adaptive): Size `numPixelsPadded * sizeof(float)`.
10. `counter`: Size `sizeof(uint32_t)`.
11. `jitter_ssh` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(float)`.
12. `inter_highest_org` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(vec3f)`.
13. `inter_highest_color` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(vec3f)`.
14. `inter_highest_alpha` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(float)`.
15. `final_highest_org` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(vec3f)`.
16. `final_highest_color` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(vec3f)`.
17. `final_highest_alpha` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(float)`.
18. `shading_color` (if SINGLE_SHADE_HEURISTIC): Size `numPixelsPadded * sizeof(vec3f)`.

Where `numPixelsPadded = next_multiple(numPixels, 256U)` and `nSamplesPerCoord = 4 * N_ITERS` if `GRADIENT_SHADING` is defined, else `N_ITERS`.

### Numbered Steps

1.  **Initialization**: Initialize ray origins and directions from the camera model. Set `alpha = 0.f` and `color = 0.f`.
2.  **Macrocell Coordinate Calculation**: Calculate macrocell coordinates for the ray origin and direction:
    $$ m_{org} = \mathbf{x}_{org} \cdot \text{macrocell\_spacings\_rcp} $$
    $$ m_{dir} = \mathbf{d} \cdot \text{macrocell\_spacings\_rcp} $$
    where `macrocell_spacings_rcp` is the reciprocal of the macrocell spacing.
3.  **DDA Iteration**: Call `dda3(m_org, m_dir, t_{min}, t_{max}, dims)` to iterate through macrocells intersecting the ray within the volume bounds $[t_{min}, t_{max}]$.
4.  **Empty Cell Check**: For each cell interval $[t_0, t_1]$, calculate the opacity upper bound $r$. If $|r| \le \epsilon$ (where $\epsilon$ is `float_epsilon`), skip the cell and proceed to the next interval [S1, unofficial].
5.  **Step Size Calculation**: Calculate the adaptive step size $ss$ using `sample_size_scaler`:
    $$ ss = \frac{t_1 - t_0}{N} $$
    $$ N = \left\lfloor \frac{t_1 - t_0}{ss_{base}} \right\rfloor + 1 $$
    where $ss_{base}$ is the base sample size determined by `adaptiveSamplingRate(step, r)` [S1, unofficial].
6.  **Ray Marching Loop**: Iterate within $[t_0, t_1]$ with step $ss$:
    ```python
    t_start = t0
    t_end = min(t1, t0 + ss)
    while t_end > t_start:
        # Perform neural inference at t_start
        # Update alpha and color using volume rendering integral
        t_start = t_end
        t_end = min(t1, t_start + ss)
    ```
7.  **Decoding Path Execution**: Depending on the mode:
    *   **Dense 3D Texture**: Decode the neural volume into a dense 3D texture first, then sample it during ray marching.
    *   **Sample Streaming**: Collect coordinates and payloads into `SampleStreamingPayload` structures. Launch host-side TCNN inference. Resume kernel with results.
    *   **In-Shader Inference**: Call fused TCNN inference directly inside the ray marching kernel. Requires `ENABLE_IN_SHADER` [S1, unofficial].

### Formulas
*   **Sample Size Scaler**:
    $$ ss = \frac{t_1 - t_0}{N}, \quad N = \left\lfloor \frac{t_1 - t_0}{ss_{base}} \right\rfloor + 1 $$
    [S1, unofficial]

### Parameters
*   **ADAPTIVE_SAMPLING**: Boolean constant (0 or 1). When 1, enables macrocell-based empty-space skipping and adaptive step lengths. When 0 or `DISABLE_ADAPTIVE_SAMPLING=ON`, falls back to fixed-step ray marching [S1, unofficial].
*   **ENABLE_IN_SHADER**: Preprocessor define. When set, allows in-shader TCNN inference. Used by `*_IN_SHADER` render modes [S1, unofficial].
*   **ENABLE_FVSRN**: Preprocessor define. When set alongside `ENABLE_IN_SHADER`, allows the in-shader branch to target the fV-SRN backend instead of TCNN. Forces `DISABLE_ADAPTIVE_SAMPLING=ON` and `KERNEL_DOUBLE_PRECISION` [S1, unofficial].
*   **MACROCELL_SIZE_MIP**: Integer (default 4). Macrocell edge length is $2^{\text{MACROCELL\_SIZE\_MIP}}$ voxels [S1, unofficial].

### Edge Cases and Pitfalls
*   **Empty Cell Handling**: If the opacity upper bound of a macrocell is zero (or near zero, $|r| \le \epsilon$), the cell is considered empty and skipped entirely to save computation [S1, unofficial].
*   **Precision**: When using `ENABLE_FVSRN`, double precision is enforced (`KERNEL_DOUBLE_PRECISION`) [S1, unofficial].
*   **Buffer Alignment**: Buffer allocations must be contiguous and pointers assigned in the specific order defined in the data structures section to ensure correct memory access in the CUDA kernel [S1, unofficial].

## Subtopics

### Photon Field Network Encoding and Architecture
A neural field method that encodes position and direction separately to predict indirect in-scattering radiance $L_i(x, \tilde{\omega})$, decoupling their effects for view-dependent phase values [S2]. It uses a 5-layer perceptron with 64 neurons per layer, ReLU activations, and no output activation [S2]. The final radiance is obtained by decoding the log-encoded prediction: $L_i = \exp(L'_i)$ [S2]. It uses 16 grid levels, 8 features per level, a 2x scaling factor between levels, and a hash table size of $2^{19}$ [S2]. Unlike most neural field methods, it produces only the indirect in-scattering term, allowing for smaller, faster networks due to lower-frequency fields [S2].

## Operating characteristics
*   **Time Complexity**: Interactive rendering speeds of 10-60fps are achieved on modern GPUs (e.g., NVIDIA RTX 3090) [S1, unofficial].
*   **Memory Complexity**: The neural representation is 10-1000x smaller than the raw volume data [S1, unofficial].
*   **Performance**: High fidelity with PSNR > 30dB [S1, unofficial].
*   **Strengths**: Interactive rendering, compact storage, support for extreme-scale data via out-of-core training [S1, unofficial].
*   **Limitations**: Requires GPU tensor cores and native CUDA framework for optimal performance [S1, unofficial].
*   **When to Use**: Real-time volume visualization, interactive exploration of large datasets [S1, unofficial].
*   **When Not to Use**: Offline rendering where pre-training is acceptable and interactivity is not required, or on hardware without tensor cores [S1, unofficial].

## Background
The method was introduced by Qi Wu, David Bauer, Michael J. Doyle, and Kwan-Liu Ma in the paper "Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation" published in IEEE Transactions on Visualization and Computer Graphics in 2023 [S1, unofficial]. It builds on the principles of [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) and aims to overcome the high cost of training and inference associated with previous neural volumetric representations [S1, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
