# Chunked Volume Rendering

Also called: VolumeRendererChunk
## Overview
Chunked Volume Rendering is a memory-management technique for image-order volume rendering that processes rays in fixed-size batches (chunks) to bound GPU memory usage during inference [S1, unofficial]. It solves the problem of out-of-memory errors when rendering high-resolution images or large ray bundles by decoupling the total ray count from the per-forward-pass memory footprint [S1, unofficial]. It differs from [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md), which optimizes sampling density during training, and [Inference Ray Compositing](3790-inference-ray-compositing.md), which focuses on the compositing logic itself; Chunked Volume Rendering specifically addresses the *batching* of the entire ray processing pipeline [S1, unofficial].
## How it works
### Inputs and Outputs
- **Inputs**:
  - `ray_bundle`: A tensor containing ray origins and directions, with shape $(N, 3)$ for origins and $(N, 3)$ for directions, where $N$ is the total number of rays [S1, unofficial].
  - `chunk_size`: An integer specifying the maximum number of rays to process in a single forward pass [S1, unofficial].
  - `sampler`: An object or function responsible for generating sample points along rays [S1, unofficial].
  - `implicit_fn`: A function (e.g., a neural radiance field query) that returns density and color at given 3D coordinates [S1, unofficial].
- **Outputs**:
  - `feature`: A concatenated tensor of rendered features (e.g., RGB, opacity) for all $N$ rays [S1, unofficial].
  - `depth`: A concatenated tensor of depth values for all $N$ rays [S1, unofficial].

### Data Structures
- `chunk_features`: A list of tensors, where each tensor holds the feature output for one chunk [S1, unofficial].
- `chunk_depths`: A list of tensors, where each tensor holds the depth output for one chunk [S1, unofficial].

### Steps
1. Initialize empty lists `chunk_features` and `chunk_depths` [S1, unofficial].
2. Determine the total number of rays $N$ from `ray_bundle.origins.shape[0]` [S1, unofficial].
3. Iterate over the range of rays in steps of `chunk_size`:
   - For each `chunk_start` in `range(0, N, self.chunk_size)`:
     a. Slice the `ray_bundle` to extract rays for the current chunk:
        $$ \text{cur\_ray\_bundle} = \text{ray\_bundle}[\text{chunk\_start} : \text{chunk\_start} + \text{self.chunk\_size}] $$
     b. Call the parent class's forward method (e.g., from [Inference Ray Marching and Integration](2148-inference-ray-marching-and-integration.md)) to process this subset:
        $$ \text{cur\_feature}, \text{cur\_depth} = \text{super().forward}(\text{sampler}, \text{implicit\_fn}, \text{cur\_ray\_bundle}) $$
     c. Append `cur_feature` to `chunk_features` and `cur_depth` to `chunk_depths` [S1, unofficial].
4. Concatenate all chunk outputs along dimension 0:
   $$ \text{feature} = \text{torch.cat}(\text{chunk\_features}, \text{dim}=0) $$
   $$ \text{depth} = \text{torch.cat}(\text{chunk\_depths}, \text{dim}=0) $$
5. Return `feature` and `depth` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward(self, sampler, implicit_fn, ray_bundle):
    # Initialize storage for chunk results
    chunk_features = []
    chunk_depths = []
    
    # Get total number of rays
    N = ray_bundle.origins.shape[0]
    
    # Process rays in chunks
    for chunk_start in range(0, N, self.chunk_size):
        # Slice the current chunk of rays
        end_index = min(chunk_start + self.chunk_size, N)
        cur_ray_bundle = ray_bundle[chunk_start:end_index]
        
        # Process the chunk using the parent implementation
        # The parent handles sampling, querying, and compositing for these rays
        cur_feature, cur_depth = super().forward(sampler, implicit_fn, cur_ray_bundle)
        
        # Store results
        chunk_features.append(cur_feature)
        chunk_depths.append(cur_depth)
        
    # Concatenate all chunk results to form the full output
    feature = torch.cat(chunk_features, dim=0)
    depth = torch.cat(chunk_depths, dim=0)
    
    return feature, depth
```

### Parameters
- `chunk_size`: Integer. The maximum number of rays processed in a single forward pass. Typical values are not explicitly defined in the primary source, but it is chosen to fit within available GPU memory [S1, unofficial].

### Edge Cases and Pitfalls
- **Memory Limits**: If `chunk_size` is too large, the forward pass may exceed GPU memory, causing an OOM error. If too small, overhead from repeated kernel launches may reduce performance [S1, unofficial].
- **Boundary Conditions**: The slicing logic must handle the last chunk, which may contain fewer rays than `chunk_size`. The pseudocode uses `min(chunk_start + self.chunk_size, N)` to ensure no out-of-bounds access [S1, unofficial].
- **Concatenation Order**: Results must be concatenated in the same order as the input rays to maintain spatial correspondence. The loop iterates sequentially, ensuring this order [S1, unofficial].

## Subtopics

### Chunked Volume Rendering with Occupancy Grid Estimator
A variant that integrates an Occupancy Grid Estimator to prune empty space before rendering [S2, unofficial]. It reshapes ray origins and view directions from $(H, W, 3)$ to $(N, 3)$, then iterates over chunks [S2, unofficial]. For each chunk, it defines a density query function (`sigma_fn`) that samples the radiance field at midpoints between `t_starts` and `t_ends` [S2, unofficial]. It calls `estimator.sampling` with parameters `near_plane`, `far_plane`, `render_step_size`, `stratified`, `cone_angle`, and `alpha_thre` to obtain occupied ray indices and depth intervals [S2, unofficial]. Finally, it composites RGB, opacity, and depth using these intervals and concatenates results [S2, unofficial]. This approach reduces computation by skipping empty regions [S2, unofficial]. Parameters include `near_plane=0.0`, `far_plane=1e10`, `render_step_size=1e-3`, `cone_angle=0.0`, `alpha_thre=0.0`, and `test_chunk_size=8192` [S2, unofficial]. During training, `chunk_size` is set to `max int32` to process all rays at once, while inference uses `test_chunk_size` [S2, unofficial].

### Tricubic B-Spline Reconstruction
An opt-in feature for chunked volume rendering that replaces linear interpolation with tricubic B-spline reconstruction to reduce texel staircase artifacts [S3, unofficial]. It requires a 2-voxel halo around chunks to function correctly [S3, unofficial]. The implementation adds constants `LINEAR_MIN_HALO` and `CUBIC_MIN_HALO` to the chunking logic [S3, unofficial]. The `NVChunkedVolume.raiseHaloTo()` method re-plans loaded volumes when the halo size changes [S3, unofficial]. Cubic interpolation is gated per draw on two conditions: the halo size must be $\ge 2$ and the volume must not be a label volume (including coarse-floor cubes) [S3, unofficial]. The parameter `volumeIsCubicInterpolation` is a boolean, defaulting to off, and is available on both backends [S3, unofficial]. The public default halo remains $[1,1,1]$ [S3, unofficial].

### Chunk 0 Eviction Fix
A bug fix for WebGL2 chunked volume rendering where the volume would blank out when chunk 0 was evicted from the cache [S3, unofficial]. The root cause was an early return in the `draw()` function when `volumeTexture` or `volumeGradientTexture` was null [S3, unofficial]. For chunked volumes, these textures are best-effort aliases of chunk 0, so evicting chunk 0 caused a silent failure [S3, unofficial]. The fix removes the early return on null textures for chunked volumes [S3, unofficial]. It also fixes `swapChunkedVolumePlan` to refresh aliases after `manager.remap()` destroys unmatched chunks, and prevents the single-volume path from deleting textures still owned by the chunked cache entry when switching from chunked to single volume [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the total number of rays, as each ray is processed exactly once [S1, unofficial]. The constant factor depends on the overhead of chunking and concatenation, which is negligible compared to the ray marching cost [S1, unofficial].
- **Memory Complexity**: $O(C \cdot M)$, where $C$ is the `chunk_size` and $M$ is the memory required to process one ray (including intermediate buffers for sampling and compositing) [S1, unofficial]. This is significantly lower than the $O(N \cdot M)$ required for unchunked rendering [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales linearly with the number of rays, but memory usage is bounded by `chunk_size` [S1, unofficial].
- **Strengths**: Prevents out-of-memory errors on high-resolution images or large ray bundles [S1, unofficial]. Allows rendering on hardware with limited VRAM [S1, unofficial].
- **Limitations**: Introduces slight overhead from chunk management and concatenation [S1, unofficial]. Requires tuning `chunk_size` for optimal performance [S1, unofficial].
- **When to use**: When rendering high-resolution volumes or large ray bundles that exceed available GPU memory [S1, unofficial].
- **When not to use**: When memory is abundant and the overhead of chunking is undesirable, or when the ray count is small enough to fit in memory without chunking [S1, unofficial].

## Background
Chunked Volume Rendering was introduced in the context of neural volume rendering pipelines to enable inference on high-resolution images [S1, unofficial]. It is part of a broader effort to make neural rendering methods practical for real-time or near-real-time applications by managing memory efficiently [S1, unofficial]. The method builds on [Inference Ray Marching and Integration](2148-inference-ray-marching-and-integration.md), which provides the core ray processing logic [S1, unofficial].

## Sources
[S1] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S2] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S3] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
