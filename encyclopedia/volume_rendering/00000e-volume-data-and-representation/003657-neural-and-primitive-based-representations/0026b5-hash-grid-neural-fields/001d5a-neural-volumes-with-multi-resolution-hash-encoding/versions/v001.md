# Neural Volumes with Multi-Resolution Hash Encoding

Also called: Hash Encoding Neural Volume, InstantVNR, Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation, VNR

## Overview
Neural Volumes with Multi-Resolution Hash Encoding (InstantVNR) is a real-time volume rendering method that represents volumetric data using a neural field decoded via multi-resolution hash encoding [S1, unofficial]. It solves the problem of high computational cost in neural volume rendering by leveraging GPU tensor cores and a native CUDA neural network framework to achieve interactive frame rates (10–60 fps) with high fidelity (PSNR > 30 dB) and compact memory usage (10–1000x smaller than raw data) [S1, unofficial]. It distinguishes itself from other neural field methods by supporting three distinct decoding paths: decoding into a dense 3D texture, sample streaming for host-side inference, and fused in-shader inference, while utilizing macro-cell acceleration for empty-space skipping [S1, unofficial].

## How it works
The method consists of a neural representation trained on volume data and a CUDA-based renderer that supports iterative ray marching with adaptive sampling.

### Inputs and Outputs
*   **Inputs:** Scene JSON containing camera parameters, volume dimensions, transfer functions, and neural network configuration (architecture, hash grid parameters).
*   **Outputs:** Rendered image (color and alpha), training metrics (loss, PSNR, SSIM), and optionally isosurface meshes.

### Data Structures

**SampleStreamingPayload**
Used for sample streaming mode. Layout:
*   `uint32_t pixel_index`
*   `float jitter`
*   `RayMarchingIter iter`
*   `union { vec3f color; vec3f org; }`
*   `float alpha`

**SingleShotPayload**
Used for single-shot heuristic mode. Layout:
*   `vec3f highest_org`
*   `vec3f highest_color`
*   `float highest_alpha`

**Ray**
*   `vec3f org`
*   `vec3f dir`
*   `float alpha` (default 0.f)
*   `vec3f color` (default 0.f, not used by shadow rays)

**Buffer Allocation (Iterative Mode)**
Buffers are allocated contiguously and pointers are assigned to `params` fields in the following order:
1.  `inference_input`
2.  `inference_output`
3.  `alpha`
4.  `color_or_org`
5.  `pixel_index`
6.  `jitter`
7.  `iter_cell` (if adaptive sampling)
8.  `iter_t_next` (if adaptive sampling)
9.  `iter_next_cell_begin`
10. `counter`
11. `jitter_ssh` (if SINGLE_SHADE_HEURISTIC)
12. `inter_highest_org`
13. `inter_highest_color`
14. `inter_highest_alpha`
15. `final_highest_org`
16. `final_highest_color`
17. `final_highest_alpha`
18. `shading_color`

Buffer sizes:
*   `numPixelsPadded * nSamplesPerCoord * sizeof(vec4f)` for inference input/output.
*   `numPixelsPadded * sizeof(SampleStreamingPayload)` for payloads.
*   `numPixelsPadded * sizeof(RayMarchingIter)` for iterators.
*   `sizeof(uint32_t)` for counter.
*   If `SINGLE_SHADE_HEURISTIC`: add `numPixelsPadded * sizeof(SingleShotPayload) * 2`, `numPixelsPadded * sizeof(vec4f)` for `shading_color`, and `numPixelsPadded * sizeof(float)` for `jitter_ssh`.

Where `numPixelsPadded = next_multiple(numPixels, 256U)` and `nSamplesPerCoord = 4 * N_ITERS` if `mode == GRADIENT_SHADING`, else `N_ITERS`.

### Algorithm Steps

1.  **Initialize Ray Marching Iterator:**
    *   If `ADAPTIVE_SAMPLING` is defined, use `dda::DDAIter`.
    *   Calculate macrocell coordinates:
        $$ m_{org} = org \times self.macrocell\_spacings\_rcp $$
        $$ m_{dir} = dir \times self.macrocell\_spacings\_rcp $$
    *   Call `dda::dda3(m_org, m_dir, tMin, tMax, dims)`.

2.  **Iterate Over Macrocells:**
    *   For each cell interval $[t_0, t_1]$:
        *   Calculate opacity upper bound $r = opacityUpperBound(self, cell)$.
        *   If $|r| \le \epsilon$ (where $\epsilon$ is `float_epsilon`), skip the cell (return true).
        *   Calculate step size $ss$:
            $$ ss = \frac{t_1 - t_0}{N} $$
            where $N = \lfloor \frac{t_1 - t_0}{ss_{base}} \rfloor + 1$, and $ss_{base}$ is the input sample size from `adaptiveSamplingRate(step, r)`.
        *   Iterate within $[t_0, t_1]$ with step $ss$:
            ```python
            t = vec2f(t0, min(t1, t0 + ss))
            while t.y > t.x:
                if not body(t):
                    return false
                t.x = t.y
                t.y = min(t.x + ss, t1)
            ```
    *   If `ADAPTIVE_SAMPLING` is not defined, fall back to fixed-step ray marching.

3.  **Neural Inference:**
    *   Depending on the rendering mode:
        *   **Decoding Path:** Decode neural volume into a dense 3D texture, then sample it.
        *   **Sample Streaming:** Collect coordinates, launch host-side TCNN inference, then resume kernel.
        *   **In-Shader:** Call fused TCNN inference directly inside the ray march (requires `ENABLE_IN_SHADER`).

### Parameters
*   `ADAPTIVE_SAMPLING`: CMake-defined boolean (0 or 1). Enables macrocell-based empty-space skipping and adaptive step lengths. When 0 or `DISABLE_ADAPTIVE_SAMPLING=ON`, uses fixed-step ray marching.
*   `ENABLE_IN_SHADER`: Preprocessor define. Enables fused in-shader TCNN inference. Required by `*_IN_SHADER` modes.
*   `ENABLE_FVSRN`: Preprocessor define. Enables fV-SRN backend instead of TCNN. Forces `DISABLE_ADAPTIVE_SAMPLING=ON` and `KERNEL_DOUBLE_PRECISION`.
*   `MACROCELL_SIZE_MIP`: Macrocell edge length is $2^{MACROCELL\_SIZE\_MIP}$ voxels. Default 4.

### Pitfalls
*   **Empty Cell Handling:** If the opacity upper bound of a macrocell is zero (or near zero, $|r| \le \epsilon$), the cell is considered empty and skipped entirely. Failure to check this leads to wasted computation on empty space.
*   **Precision:** When using `ENABLE_FVSRN`, double precision is forced.

## Subtopics

### Photon Field Network Encoding and Architecture
A neural field method that encodes position and direction separately to predict indirect in-scattering radiance $L_i(x, \tilde{\omega})$, decoupling their effects for view-dependent phase values [S2]. It uses a 5-layer perceptron with 64 neurons per layer, ReLU activations, and no output activation [S2]. The final radiance is obtained by decoding the log-encoded prediction $L'_i(x, \tilde{\omega})$ using $L_i = \exp(L'_i)$ [S2]. It uses 16 grid levels, 8 features per level, a 2x scaling factor between levels, and a hash table size of $2^{19}$ [S2]. The phase coefficient $g$ is left untouched (unitless scale) and not encoded [S2]. Inputs are 6-dimensional (position + direction), separately encoded using hashgrid method before passing to the MLP [S2]. Unlike most neural field methods that contribute to the full image appearance, photon fields produce only the indirect in-scattering term, allowing for smaller, faster networks due to lower-frequency fields [S2].

## Operating characteristics
*   **Time Complexity:** Interactive frame rates (10–60 fps) are achieved by leveraging GPU tensor cores and macro-cell acceleration [S1, unofficial].
*   **Memory Complexity:** Compact representation, 10–1000x smaller than raw volume data [S1, unofficial].
*   **Performance:** High fidelity with PSNR > 30 dB [S1, unofficial].
*   **Scaling:** Supports extreme-scale volume data via an efficient out-of-core training strategy, potentially scaling to terascale using an NVIDIA RTX 3090 workstation [S1, unofficial].
*   **Strengths:** Real-time rendering, high fidelity, compact storage, out-of-core training support.
*   **Limitations:** Requires CUDA-capable GPU, specific CMake flags for features like in-shader inference.
*   **When to use:** Interactive visualization of large volumetric datasets where real-time feedback is needed.
*   **When not to use:** Offline rendering where pre-training is acceptable and real-time is not required, or on hardware without CUDA support.

## Background
The method was introduced by Qi Wu, David Bauer, Michael J. Doyle, and Kwan-Liu Ma in the paper "Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation" published in IEEE Transactions on Visualization and Computer Graphics in 2023 [S1, unofficial]. It builds on the concept of neural fields and hash grid encoding [S1, unofficial]. The project is available on GitHub as `VIDILabs/instantvnr` [S1, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
