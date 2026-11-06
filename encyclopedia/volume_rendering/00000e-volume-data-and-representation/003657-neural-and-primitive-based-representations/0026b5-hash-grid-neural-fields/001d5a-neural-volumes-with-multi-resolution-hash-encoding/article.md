# Neural Volumes with Multi-Resolution Hash Encoding

Also called: Hash Encoding Neural Volume, InstantVNR, Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation, VNR

## Overview
Neural Volumes with Multi-Resolution Hash Encoding (InstantVNR) is a method for interactive volume rendering that compresses volumetric data into a neural representation using multi-resolution hash grids [S1, unofficial]. It solves the high inference cost of traditional neural fields by leveraging GPU tensor cores and a native CUDA neural network framework (TCNN) to achieve 10–60 fps rendering [S1, unofficial]. Unlike [HashGrid Encoding](3866-hashgrid-encoding.md) which focuses on the encoding mechanism itself, InstantVNR provides a complete rendering pipeline including macro-cell acceleration for empty-space skipping and three distinct decoding paths (dense texture, sample streaming, and in-shader inference) to support real-time interaction [S1, unofficial].

## How it works
The method defines a scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ using a neural network parameterized by $\Theta$. The input is a continuous coordinate vector (typically 3D spatial position $\mathbf{x}$, potentially augmented with viewing direction $\mathbf{d}$), and the output is a signal vector (e.g., density $\sigma$ and color $\mathbf{c}$) [S1, unofficial]. The network is trained by minimizing a loss between rendered observations and ground truth images [S1, unofficial].

### Data Structures
The implementation relies on specific CUDA device-side structures for ray marching and payload management [S1, unofficial]:

*   **Ray**:
    *   `vec3f org`: Ray origin.
    *   `vec3f dir`: Ray direction.
    *   `float alpha`: Default `0.f`.
    *   `vec3f color`: Default `0.f` (not used by shadow rays).
*   **SampleStreamingPayload**:
    *   `uint32_t pixel_index`
    *   `float jitter`
    *   `RayMarchingIter iter`
    *   `union { vec3f color; vec3f org; }`
    *   `float alpha`
*   **SingleShotPayload**:
    *   `vec3f highest_org`
    *   `vec3f highest_color`
    *   `float highest_alpha`

### Buffer Allocation (Iterative Mode)
For iterative rendering modes, a contiguous buffer is allocated to store inference inputs, outputs, and ray marching state [S1, unofficial].

1.  Calculate `numPixelsPadded` as `next_multiple(numPixels, 256U)`.
2.  Calculate `nSamplesPerCoord`:
    *   If `mode == GRADIENT_SHADING`: `4 * N_ITERS`.
    *   Else: `N_ITERS`.
3.  Calculate base buffer size: `numPixelsPadded * nSamplesPerCoord * sizeof(vec4f)` for inference input/output.
4.  Add `numPixelsPadded * sizeof(SampleStreamingPayload)` for payloads.
5.  Add `numPixelsPadded * sizeof(RayMarchingIter)` for iterators.
6.  Add `sizeof(uint32_t)` for the counter.
7.  If `mode == SINGLE_SHADE_HEURISTIC`:
    *   Add `numPixelsPadded * sizeof(SingleShotPayload) * 2`.
    *   Add `numPixelsPadded * sizeof(vec4f)` for `shading_color`.
    *   Add `numPixelsPadded * sizeof(float)` for `jitter_ssh`.
8.  Assign pointers to `params` fields in order: `inference_input`, `inference_output`, `alpha`, `color_or_org`, `pixel_index`, `jitter`, `iter_cell` (if adaptive), `iter_t_next` (if adaptive), `iter_next_cell_begin`, `counter`, `jitter_ssh` (if SSH), `inter_highest_org`, `inter_highest_color`, `inter_highest_alpha`, `final_highest_org`, `final_highest_color`, `final_highest_alpha`, `shading_color` [S1, unofficial].

### Ray Marching with Adaptive Sampling
The renderer supports adaptive sampling via macro-cells to skip empty space [S1, unofficial].

1.  **Macrocell Coordinates**:
    *   `m_org = org * self.macrocell_spacings_rcp`
    *   `m_dir = dir * self.macrocell_spacings_rcp`
2.  **DDA Iteration**: Call `dda::dda3(m_org, m_dir, tMin, tMax, dims)` to get cell intervals $[t_0, t_1]$ [S1, unofficial].
3.  **Empty Cell Check**: For each cell, calculate opacity upper bound $r = \text{opacityUpperBound}(\text{self}, \text{cell})$. If $\text{fabsf}(r) \le \text{float\_epsilon}$, skip the cell (return true) [S1, unofficial].
4.  **Step Size Calculation**:
    *   Calculate base step size $ss_{base} = \text{adaptiveSamplingRate}(\text{step}, r)$.
    *   Calculate number of steps $N = \lfloor \frac{t_1 - t_0}{ss_{base}} \rfloor + 1$.
    *   Calculate actual step size $ss = \frac{t_1 - t_0}{N}$ [S1, unofficial].
5.  **Marching Loop**:
    *   Initialize $t = (t_0, \min(t_1, t_0 + ss))$.
    *   While $t.y > t.x$:
        *   Execute body for interval $[t.x, t.y]$.
        *   Update $t.x = t.y$.
        *   Update $t.y = \min(t.x + ss, t_1)$ [S1, unofficial].

### Decoding Paths
The method supports three distinct paths for decoding the neural representation [S1, unofficial]:
1.  **Decoding**: Decodes the neural volume into a dense 3D texture first, then samples it (similar to [SimpleVolume](3850-simple-volume.md)).
2.  **Sample Streaming**: Collects sample coordinates, launches host-side TCNN inference, and resumes the kernel.
3.  **In-Shader Inference**: Calls fused TCNN inference directly inside the ray march kernel. Requires `ENABLE_IN_SHADER` [S1, unofficial].

### Pseudocode
The following pseudocode implements the `sample_size_scaler` and the core adaptive stepping logic described in the sources [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_size_scaler(ss_base: float, t0: float, t1: float) -> float:
    """
    Calculates the actual step size for adaptive sampling.
    ss_base: The input sample size (adaptiveSamplingRate(step, r)).
    t0, t1: The start and end of the current cell interval.
    """
    # N = floor((t1 - t0) / ss_base) + 1
    # In integer arithmetic for positive values:
    N = int((t1 - t0) / ss_base) + 1
    return (t1 - t0) / N

def ray_march_adaptive(org, dir, tMin, tMax, dims, macrocell_spacings_rcp, opacityUpperBound, body):
    """
    Simulates the adaptive ray marching loop with macro-cell acceleration.
    """
    float_epsilon = 1e-6
    
    # 1. Calculate macrocell coordinates
    m_org = org * macrocell_spacings_rcp
    m_dir = dir * macrocell_spacings_rcp
    
    # 2. Get cell intervals using DDA
    # dda3 returns a list of intervals [t0, t1] for each macrocell
    intervals = dda3(m_org, m_dir, tMin, tMax, dims)
    
    for t0, t1 in intervals:
        # 3. Check for empty cells
        r = opacityUpperBound(macrocell_spacings_rcp, t0, t1) # Simplified signature for illustration
        if abs(r) <= float_epsilon:
            continue
            
        # 4. Calculate adaptive step size
        ss_base = adaptiveSamplingRate(step, r)
        ss = sample_size_scaler(ss_base, t0, t1)
        
        # 5. Iterate within the cell
        t_x = t0
        t_y = min(t1, t0 + ss)
        
        while t_y > t_x:
            # Execute body for the segment [t_x, t_y]
            body(t_x, t_y)
            
            # Advance
            t_x = t_y
            t_y = min(t_x + ss, t1)
```

## Subtopics

### Photon Field Network Encoding and Architecture
A neural field method that uses multi-resolution hash encoding to predict indirect in-scattering radiance $L_i(x, \tilde{\omega})$ [S2]. It encodes position and direction separately to decouple their effects for view-dependent phase values [S2].
*   **Architecture**: 5-layer perceptron with 64 neurons per layer, ReLU activations, no output activation [S2].
*   **Encoding**: 16 grid levels, 8 features per level, 2x scaling factor, hash table size $2^{19}$ [S2].
*   **Output**: Predicts log-encoded radiance $L'_i$, decoded via $L_i = \exp(L'_i)$ [S2].
*   **Usage**: Produces only the indirect in-scattering term, allowing for smaller, faster networks due to lower-frequency fields compared to full appearance methods [S2].

## Operating characteristics
*   **Performance**: Achieves interactive frame rates of 10–60 fps [S1, unofficial].
*   **Fidelity**: High-fidelity reconstruction with PSNR > 30dB [S1, unofficial].
*   **Compression**: Model sizes are 10–1000x smaller than the original volume data [S1, unofficial].
*   **Hardware**: Requires modern GPU tensor cores and a native CUDA neural network framework (TCNN) [S1, unofficial].
*   **Scaling**: Supports extreme-scale data via an out-of-core training strategy, potentially scaling to terascale volumes on an NVIDIA RTX 3090 workstation [S1, unofficial].
*   **Training**: Can fit the entire training step inside the rendering loop, skipping pre-training [S1, unofficial].
*   **Limitations**: Requires `ENABLE_IN_SHADER` for in-shader inference modes; `ENABLE_FVSRN` forces `DISABLE_ADAPTIVE_SAMPLING` [S1, unofficial].

## Background
The method was introduced by Qi Wu, David Bauer, Michael J. Doyle, and Kwan-Liu Ma in the paper "Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation" [S1, unofficial]. It was published in IEEE Transactions on Visualization and Computer Graphics in 2023 (arXiv preprint 2207.11620) [S1, unofficial]. The work demonstrates that neural volumetric representations can be made interactive by combining multi-resolution hash encoding with GPU-accelerated inference and macro-cell ray marching acceleration [S1, unofficial].

## Sources
[S1] Interactive Volume Visualization Via Multi-Resolution Hash Encoding Based Neural Representation, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
