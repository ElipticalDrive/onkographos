# Neural Volumes with Multi-Resolution Hash Encoding

Also called: Hash Encoding Neural Volume, InstantVNR, Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation, VNR

## Overview
Neural Volumes with Multi-Resolution Hash Encoding (InstantVNR) is a method for interactive volume rendering that compresses volumetric data into a neural representation using multi-resolution hash grids [S1, unofficial]. It solves the high inference cost of traditional neural fields by leveraging GPU tensor cores and a native CUDA neural network framework (TCNN) to achieve 10–60 fps rendering [S1, unofficial]. Unlike [HashGrid Encoding](3866-hashgrid-encoding.md) which focuses on the encoding mechanism itself, InstantVNR provides a complete rendering pipeline including macro-cell acceleration for empty-space skipping and three distinct decoding paths (dense texture, sample streaming, and in-shader inference) to support real-time interaction [S1, unofficial].

## How it works
Builds on the shared principles of hash grid neural fields [3855-hash-grid-neural-fields.md], the method defines a scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ [G]. The input is a continuous coordinate vector (typically 3D spatial position $\mathbf{x}$, potentially augmented with viewing direction $\mathbf{d}$), and the output is a signal vector (e.g., density $\sigma$ and color $\mathbf{c}$) [G]. The network is trained by minimizing a loss between rendered observations and ground truth images [G].

### Data Structures
The implementation relies on specific CUDA device-side structures for ray marching and payload management [S1, unofficial]:
* **Ray**: `vec3f org`, `vec3f dir`, `float alpha` (default `0.f`), `vec3f color` (default `0.f`, not used by shadow rays) [S1, unofficial].
* **SampleStreamingPayload**: `uint32_t pixel_index`, `float jitter`, `RayMarchingIter iter`, `union { vec3f color; vec3f org; }`, `float alpha` [S1, unofficial].
* **SingleShotPayload**: `vec3f highest_org`, `vec3f highest_color`, `float highest_alpha` [S1, unofficial].

### Buffer Allocation (Iterative Mode)
For iterative rendering modes, a contiguous buffer is allocated to store inference inputs, outputs, and ray marching state [S1, unofficial].
1. Calculate `numPixelsPadded` as `next_multiple(numPixels, 256U)` [S1, unofficial].
2. Calculate `nSamplesPerCoord`: `4 * N_ITERS` if `mode == GRADIENT_SHADING`, else `N_ITERS` [S1, unofficial].
3. Calculate base buffer size: `numPixelsPadded * nSamplesPerCoord * sizeof(vec4f)` for inference input/output [S1, unofficial].
4. Add `numPixelsPadded * sizeof(SampleStreamingPayload)` for payloads [S1, unofficial].
5. Add `numPixelsPadded * sizeof(RayMarchingIter)` for iterators [S1, unofficial].
6. Add `sizeof(uint32_t)` for the counter [S1, unofficial].
7. If `mode == SINGLE_SHADE_HEURISTIC`: add `numPixelsPadded * sizeof(SingleShotPayload) * 2`, `numPixelsPadded * sizeof(vec4f)` for `shading_color`, and `numPixelsPadded * sizeof(float)` for `jitter_ssh` [S1, unofficial].
8. Assign pointers to `params` fields in order: `inference_input`, `inference_output`, `alpha`, `color_or_org`, `pixel_index`, `jitter`, `iter_cell` (if adaptive), `iter_t_next` (if adaptive), `iter_next_cell_begin`, `counter`, `jitter_ssh` (if SSH), `inter_highest_org`, `inter_highest_color`, `inter_highest_alpha`, `final_highest_org`, `final_highest_color`, `final_highest_alpha`, `shading_color` [S1, unofficial].

### Ray Marching with Adaptive Sampling
The renderer supports adaptive sampling via macro-cells to skip empty space [S1, unofficial].
1. **Macrocell Coordinates**: `m_org = org * self.macrocell_spacings_rcp` and `m_dir = dir * self.macrocell_spacings_rcp` [S1, unofficial].
2. **DDA Iteration**: Call `dda::dda3(m_org, m_dir, tMin, tMax, dims)` to get cell intervals $[t_0, t_1]$ [S1, unofficial].
3. **Empty Cell Check**: For each cell, calculate opacity upper bound $r = \text{opacityUpperBound}(\text{self}, \text{cell})$. If $\text{fabsf}(r) \le \text{float\_epsilon}$, skip the cell (return true) [S1, unofficial].
4. **Step Size Calculation**: Calculate base step size $ss_{base} = \text{adaptiveSamplingRate}(\text{step}, r)$. Calculate number of steps $N = \lfloor \frac{t_1 - t_0}{ss_{base}} \rfloor + 1$. Calculate actual step size $ss = \frac{t_1 - t_0}{N}$ [S1, unofficial].
5. **Marching Loop**: Initialize $t = (t_0, \min(t_1, t_0 + ss))$. While $t.y > t.x$: execute body for interval $[t.x, t.y]$, update $t.x = t.y$, update $t.y = \min(t.x + ss, t_1)$ [S1, unofficial].

### Decoding Paths
The method supports three distinct paths for decoding the neural representation [S1, unofficial]:
1. **Decoding**: Decodes the neural volume into a dense 3D texture first, then samples it (similar to [SimpleVolume](3850-simple-volume.md)) [S1, unofficial].
2. **Sample Streaming**: Collects sample coordinates, launches host-side TCNN inference, and resumes the kernel [S1, unofficial].
3. **In-Shader Inference**: Calls fused TCNN inference directly inside the ray march kernel. Requires `ENABLE_IN_SHADER` [S1, unofficial].

### Pseudocode
The following pseudocode implements the `sample_size_scaler` and the core adaptive stepping logic described in the sources [S1, unofficial]. It includes a standard 3D DDA to compute intervals and a basic accumulation body to ensure correctness.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_size_scaler(ss_base: float, t0: float, t1: float) -> float:
    """Calculates the actual step size for adaptive sampling."""
    if ss_base <= 0:
        return t1 - t0
    N = math.floor((t1 - t0) / ss_base) + 1
    return (t1 - t0) / N

def compute_intervals(org, dir, tMin, tMax):
    """Standard 3D DDA for an axis-aligned bounding box."""
    t_near = []
    t_far = []
    for i in range(3):
        if dir[i] == 0:
            if org[i] < tMin[i] or org[i] > tMax[i]:
                return []
            t_near.append(float('-inf'))
            t_far.append(float('inf'))
        else:
            t1 = (tMin[i] - org[i]) / dir[i]
            t2 = (tMax[i] - org[i]) / dir[i]
            t_near.append(min(t1, t2))
            t_far.append(max(t1, t2))
    t_entry = max(t_near)
    t_exit = min(t_far)
    if t_entry > t_exit or t_exit < 0:
        return []
    return [(t_entry, t_exit)]

def ray_march_adaptive(org, dir, tMin, tMax, ss_base, opacity_upper_bound, body_func):
    """Simulates the adaptive ray marching loop with macro-cell acceleration."""
    float_epsilon = 1e-6
    intervals = compute_intervals(org, dir, tMin, tMax)
    for t0, t1 in intervals:
        r = opacity_upper_bound(t0, t1)
        if abs(r) <= float_epsilon:
            continue
        ss = sample_size_scaler(ss_base, t0, t1)
        t_x = t0
        t_y = min(t1, t0 + ss)
        while t_y > t_x:
            body_func(t_x, t_y)
            t_x = t_y
            t_y = min(t_x + ss, t1)
```

## Subtopics

### Photon Field Network Encoding and Architecture
A neural field method that uses multi-resolution hash encoding to predict indirect in-scattering radiance $L_i(x, \tilde{\omega})$ [S2]. It encodes position and direction separately to decouple their effects for view-dependent phase values [S2].
* **Architecture**: 5-layer perceptron with 64 neurons per layer, ReLU activations, no output activation [S2].
* **Encoding**: 16 grid levels, 8 features per level, 2x scaling factor, hash table size $2^{19}$ [S2].
* **Output**: Predicts log-encoded radiance $L'_i$, decoded via $L_i = \exp(L'_i)$ [S2].
* **Usage**: Produces only the indirect in-scattering term, allowing for smaller, faster networks due to lower-frequency fields compared to full appearance methods [S2].

## Operating characteristics
* **Performance**: Achieves interactive frame rates of 10–60 fps [S1, unofficial].
* **Fidelity**: High-fidelity reconstruction with PSNR > 30dB [S1, unofficial].
* **Compression**: Model sizes are 10–1000x smaller than the original volume data [S1, unofficial].
* **Hardware**: Requires modern GPU tensor cores and a native CUDA neural network framework (TCNN) [S1, unofficial].
* **Scaling**: Supports extreme-scale data via an out-of-core training strategy, potentially scaling to terascale volumes on an NVIDIA RTX 3090 workstation [S1, unofficial].
* **Training**: Can fit the entire training step inside the rendering loop, skipping pre-training [S1, unofficial].
* **Limitations**: Requires `ENABLE_IN_SHADER` for in-shader inference modes; `ENABLE_FVSRN` forces `DISABLE_ADAPTIVE_SAMPLING` [S1, unofficial].

## Background
The method was introduced by Qi Wu, David Bauer, Michael J. Doyle, and Kwan-Liu Ma in the paper "Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation" [S1, unofficial]. It was published in IEEE Transactions on Visualization and Computer Graphics in 2023 (arXiv preprint 2207.11620) [S1, unofficial]. The work demonstrates that neural volumetric representations can be made interactive by combining multi-resolution hash encoding with GPU-accelerated inference and macro-cell ray marching acceleration [S1, unofficial].

## Sources
[S1] Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
