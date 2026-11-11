# Chunked Volume Rendering

Also called: Chunked Volume Rendering with Early Stopping, Memory-efficient chunked rendering, TBRayMarcher Plugin, VolumeRendererChunk

## Overview
Chunked volume rendering is a memory-efficient inference technique that processes sampled points along rays in fixed-size batches (chunks) to manage GPU memory usage, allowing for trade-offs between time and memory [S2, unofficial]. It solves the problem of rendering large volumetric datasets that exceed available VRAM by breaking the ray integration process into manageable segments [S1, S2, unofficial]. Unlike standard ray marching which processes all samples for all rays simultaneously, chunked rendering iterates through rays in groups, accumulating results incrementally [S2, unofficial]. This method is distinct from sparse ray marching (which prunes empty space) and light-space opacity precomputation (which pre-calculates lighting), focusing instead on batching the evaluation and integration steps [S2, unofficial].

## How it works
The method processes a batch of rays by dividing the total number of rays or samples into chunks of a specified size. It maintains global accumulation variables across chunks to ensure the final composite is correct, while resetting local state for each chunk.

### Inputs and Outputs
*   **Inputs**:
    *   `ray_bundle`: Contains ray origins and directions. Shape: $(N, 3)$ for origins, $(N, 3)$ for directions, where $N$ is the total number of rays.
    *   `implicit_fn`: A function or neural network that evaluates density and color (or features) at given 3D coordinates.
    *   `sampler`: A sampler object that generates sample points along the rays.
    *   `chunk_size`: The number of rays (or samples) to process in a single forward pass.
    *   `tolerance`: (Optional) Threshold for early stopping.
*   **Outputs**:
    *   `feature`: Rendered feature tensor (e.g., color, depth) concatenated from all chunks. Shape: $(N, \text{channels})$.
    *   `depth`: Rendered depth tensor. Shape: $(N, 1)$.

### Data Structures
*   `accumulated_free_energy`: Tensor of shape $(N,)$, initialized to 0. Stores the cumulative optical depth (free energy) for each ray.
*   `accumulated_evaluations`: Tensor of shape $(N,)$, initialized to 0. Stores the cumulative weight (opacity) contributed by rendered samples.
*   `chunk_features`: List of tensors, each of shape $(\text{chunk\_size}, \text{channels})$.
*   `chunk_depths`: List of tensors, each of shape $(\text{chunk\_size}, 1)$.

### Parameters
*   `chunk_size`: Default 65536 ($1024 \times 64$). Controls the number of rays processed per forward pass [S2, unofficial].
*   `valid_chunk_size`: Used during inference. Defaults to `chunk_size // 1024` if not specified, otherwise equals `chunk_size` [S2, unofficial].
*   `raymarching_tolerance`: Default 0.0. If $> 0$, rays are terminated when accumulated free energy exceeds $-\log(\text{tolerance})$ [S2, unofficial].

### Formulas
The rendering follows standard volume rendering equations, computed per chunk and accumulated:

1.  **Free Energy (Optical Depth)**:
    Let $\sigma_t$ be the density at sample $t$. The free energy is often derived from density.
    $$ \text{free\_energy}_t = \sigma_t \cdot \Delta t $$
    The probability of empty space up to step $t$ is:
    $$ b_t = \exp\left(-\sum_{i=1}^{t} \text{shifted\_free\_energy}_i\right) $$
    where $\text{shifted\_free\_energy}$ is $\text{free\_energy}$ shifted by one step (to handle the first sample correctly) [S2, unofficial].

2.  **Hit Probability**:
    The probability of a hit at step $t$ is:
    $$ a_t = 1 - \exp(-\text{free\_energy}_t) $$
    The final probability (weight) for sample $t$ is:
    $$ \text{probs}_t = a_t \times b_t $$

3.  **Accumulation**:
    *   Accumulated Free Energy:
        $$ \text{accumulated\_free\_energy} \leftarrow \text{accumulated\_free\_energy} + \sum_{t \in \text{chunk}} \text{free\_energy}_t $$
    *   Accumulated Evaluations (Opacity):
        $$ \text{accumulated\_evaluations} \leftarrow \text{accumulated\_evaluations} + \sum_{t \in \text{chunk}} \text{probs}_t $$

4.  **Final Render**:
    *   Rendered Color:
        $$ \text{color} = \sum_{t} \text{texture}_t \times \text{probs}_t $$
    *   Rendered Depth:
        $$ \text{depth} = \sum_{t} \text{sampled\_depth}_t \times \text{probs}_t $$
    *   Missed Probability:
        $$ \text{missed} = 1 - \sum_{t} \text{probs}_t $$

### Steps
1.  Initialize `accumulated_free_energy` to zeros of shape $(N,)$.
2.  Initialize `accumulated_evaluations` to zeros of shape $(N,)$.
3.  Initialize empty lists `chunk_features` and `chunk_depths`.
4.  Determine the number of rays $N$ from `ray_bundle.origins.shape[0]`.
5.  Loop over chunks: for `chunk_start` in `range(0, N, chunk_size)`:
    a.  Slice the ray bundle: `cur_ray_bundle = ray_bundle[chunk_start : chunk_start + chunk_size]`.
    b.  Get the number of rays in this chunk, $N_c$.
    c.  Call the parent forward method or `forward_once` with `sampler`, `implicit_fn`, and `cur_ray_bundle`.
        *   If `forward_once` returns `None` (e.g., no samples hit the voxel grid), append zero-filled tensors of shape $(N_c, \text{channels})$ to `chunk_features` and $(N_c, 1)$ to `chunk_depths`. Continue to next chunk.
    d.  Let `outputs` be the result. If `outputs` is not `None`:
        i.  Extract `free_energy` from `outputs`.
        ii. Update `accumulated_free_energy`:
            $$ \text{accumulated\_free\_energy}[\text{chunk\_start} : \text{chunk\_start} + N_c] \leftarrow \text{accumulated\_free\_energy}[\dots] + \sum_{t} \text{free\_energy}_t $$
        iii. Extract `texture` and `sampled_depth` from `outputs`.
        iv. Compute `probs` using the formulas above (using the *current* chunk's free energy and the *accumulated* free energy from previous chunks).
        v. Compute `color` and `depth` for the chunk.
        vi. Append `color` to `chunk_features` and `depth` to `chunk_depths`.
    e.  **Early Stopping**: If `tolerance > 0`:
        i.  Identify rays where `accumulated_free_energy > -log(tolerance)`.
        ii. Zero out the contributions for these rays in the current chunk (or skip further processing for them if the sampler supports it).
6.  Concatenate all tensors in `chunk_features` along dimension 0 to form `feature`.
7.  Concatenate all tensors in `chunk_depths` along dimension 0 to form `depth`.
8.  Return `feature` and `depth`.

### Edge Cases and Pitfalls
*   **No Hits**: If `sample_mask.sum() == 0` (no samples hit the voxel grid), return `None` and 0 evaluations to skip computation [S2, unofficial].
*   **Shape Consistency**: If `forward_once` returns `None`, zero-filled tensors must be appended to maintain shape consistency for the final concatenation [S2, unofficial].
*   **Precision**: Accumulating free energy over many chunks can lead to precision issues if not handled carefully. Using double precision for accumulation variables may be necessary for very large volumes [G].
*   **Chunk Borders**: In some implementations (e.g., UE plugins), chunking data into smaller volumes can introduce visual artifacts at chunk borders because lighting is computed independently per chunk [S1, unofficial].

## Subtopics

### Chunked Volume Rendering with Occupancy Grid Estimator
A variant that uses an `OccGridEstimator` to sample along rays, querying a radiance field for density and color at sample midpoints, and compositing the result per chunk [S4, unofficial]. It reshapes ray origins and viewdirs from 3D $(H, W, 3)$ to 2D $(N, 3)$, sets chunk size to `max int32` during training or `test_chunk_size` (default 8192) otherwise [S4, unofficial]. For each chunk, it defines a density function `sigma_fn` to query the radiance field at positions `origins + dirs * (t_starts + t_ends) / 2.0`, optionally passing timestamps for temporal scenes [S4, unofficial]. It calls `estimator.sampling` with parameters `near_plane` (default 0.0), `far_plane` (default $1e10$), `render_step_size` (default $1e-3$), `stratified` flag, `cone_angle` (default 0.0), and `alpha_thre` (default 0.0) to obtain `ray_indices`, `t_starts`, `t_ends` [S4, unofficial]. Then it calls `rendering` with these values to composite RGB, opacity, depth, and extras [S4, unofficial]. Results are concatenated along dimension 0 and reshaped to original image dimensions [S4, unofficial].

### Tricubic B-Spline Reconstruction
An opt-in reconstruction method to fix texel staircase artifacts in chunked volumes, though it does not remove ringing [S5, unofficial]. It requires a 2-voxel halo [S5, unofficial]. Implementation involves defining `LINEAR_MIN_HALO` and `CUBIC_MIN_HALO` in `chunking.ts`, and `NVChunkedVolume.raiseHaloTo()` re-plans loaded volumes when the halo setter fires [S5, unofficial]. Both views gate cubic interpolation per draw on halo $\ge 2$ and for non-label volumes (including coarse-floor cubes) [S5, unofficial]. The parameter `volumeIsCubicInterpolation` is a boolean, default off, available on both backends [S5, unofficial]. The public default halo stays $[1,1,1]$ [S5, unofficial].

### Chunk 0 Eviction Fix
A fix for WebGL2 volume blanking when chunk 0 is evicted [S5, unofficial]. The issue was caused by `draw()` early-returning on null `volumeTexture`/`volumeGradientTexture`, which were best-effort aliases of chunk 0 [S5, unofficial]. The fix involves: 1. Removing the early-return on null textures for chunked volumes. 2. Fixing `swapChunkedVolumePlan` to refresh aliases after `manager.remap()` destroys unmatched chunks. 3. Fixing the switch from chunked to single volume to prevent the single-volume path from deleting textures the chunked cache entry still owns [S5, unofficial].

### 4-in-1 Material Workaround
A workaround for lighting artifacts at chunk borders in Unreal Engine by hacking the Material of the Heterogeneous Volume Actor (HVA) [S6, unofficial]. It extends the single SVT evaluation to select and sample from four different chunks, feeding the internal lighting cache with values from all chunks to create seamless rendering [S6, unofficial]. It uses an SVT of full resolution spanning the volume with values only at min/max corners to trick the HVA into believing it has one volume, while sampling four payload chunks as if they were one texture [S6, unofficial]. This results in a heavy performance impact (~1fps) [S6, unofficial]. Performance is compensated by allowing interactive navigation with fewer samples per ray march (e.g., 32) and rendering with more samples (e.g., 512) when an interesting view is found [S6, unofficial]. The console command `r.HeterogeneousVolumes.MaxStepCount=32` sets the max step count for interactivity [S6, unofficial].

### Partitioned Volume Rendering Derivation
A derivation for distributed volume rendering where the global composition is split into per-segment accumulations [S7]. The accumulated color is decomposed as:
$$ C(t_1 \rightarrow t_{N+1}) = \sum_{k=1}^N T(t_1 \rightarrow t_k) C(t_k \rightarrow t_{k+1}) $$
where $C(t_k \rightarrow t_{k+1}) = \int_{t_k}^{t_{k+1}} T(t_k \rightarrow t) \sigma(t) c(t) dt$ [S7]. The distortion loss is decomposed as:
$$ \mathcal{L}_{dist}(t_1 \rightarrow t_{N+1}) = 2 \sum_{k=1}^N T(t_1 \rightarrow t_k) S(t_1 \rightarrow t_k) + \sum_{k=1}^N T(t_1 \rightarrow t_k)^2 \mathcal{L}_{dist}(t_k \rightarrow t_{k+1}) $$
where $S(t_1 \rightarrow t_k) = D(t_k \rightarrow t_{k+1})A(t_1 \rightarrow t_k) - A(t_k \rightarrow t_{k+1})D(t_1 \rightarrow t_k)$, with $A$ being accumulated weight and $D$ accumulated depth [S7]. $N$ is the number of segments (partitions) along the ray [S7].

### Surface Volume Rendering
A rendering method that renders at the estimated intersection point of the ray with the surface, rather than integrating along the entire ray [S2, unofficial]. It computes normalized probabilities by clamping min $1e-6$ and masking out non-hit voxels [S2, unofficial]. The intersection depth is:
$$ \text{n\_depth} = \frac{\sum_t \text{sampled\_depth}_t \times \text{n\_probs}_t}{\sum_t \text{n\_probs}_t} $$
The boundary depth is:
$$ \text{n\_bound} = \text{sampled\_depth} + \frac{\text{sampled\_distance}}{2} $$
The voxel index at intersection is determined by counting boundaries $\le \text{n\_depth}$ [S2, unofficial]. It calls `forward_once` with new samples to get texture and normal at the intersection [S2, unofficial]. The rendered color is:
$$ \text{color} = \text{texture} \times (1 - \text{missed}) $$
An eikonal term for surface rendering is:
$$ \text{eikonal} = (\|\text{normal}\|_2 - 1)^2 $$

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S / C)$, where $N$ is the number of rays, $S$ is the number of samples per ray, and $C$ is the chunk size. The overhead of chunking is minimal if $C$ is large enough to amortize kernel launch costs.
*   **Memory Complexity**: $O(C \cdot S \cdot D)$, where $D$ is the dimensionality of the features. This allows rendering scenes with $N \gg C$ by keeping GPU memory usage constant regardless of the total number of rays or samples.
*   **Measured Performance**: In UE5, chunked rendering with the 4-in-1 workaround achieved ~10-20 fps for large datasets, but the seamless lighting workaround dropped framerates to ~1fps [S1, S6, unofficial]. Standard chunked rendering without seamless lighting is faster but has artifacts [S1, unofficial].
*   **Scaling**: Scales well with chunk size; larger chunks reduce overhead but increase memory usage. The method is limited by VRAM fragmentation and API limits (e.g., DirectX 11) for very large datasets [S1, unofficial].
*   **Strengths**: Enables rendering of datasets larger than VRAM; memory-efficient; supports early stopping.
*   **Limitations**: Chunk borders may have lighting artifacts if not handled specially; early stopping may introduce bias if not careful; overhead of managing chunks.
*   **When to use**: When VRAM is insufficient for full ray marching; when real-time inference is required for large volumes.
*   **When not to use**: When high-quality seamless lighting across chunks is required without performance penalties; when the dataset fits comfortably in VRAM.

## Background
Chunked volume rendering originated as a solution for rendering large medical and scientific datasets in game engines and real-time applications [S1, unofficial]. The TBRayMarcher plugin, initially for medical data, used efficient illumination caching and true volume textures, but struggled with large datasets due to VRAM fragmentation [S1, unofficial]. The technique was further developed in the context of Neural Volume Rendering (NVR) and NeRFs, where memory efficiency is critical for training and inference on large scenes [S2, S4, unofficial]. Variants like occupancy grid-based chunking and partitioned rendering for distributed NeRFs have extended the method to handle even larger scales [S4, unofficial][S7].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.emergentmind.com/papers/2504.07485, fetched 2026-10-26 (unofficial)
[S2] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S4] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S5] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
[S6] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
[S7] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
