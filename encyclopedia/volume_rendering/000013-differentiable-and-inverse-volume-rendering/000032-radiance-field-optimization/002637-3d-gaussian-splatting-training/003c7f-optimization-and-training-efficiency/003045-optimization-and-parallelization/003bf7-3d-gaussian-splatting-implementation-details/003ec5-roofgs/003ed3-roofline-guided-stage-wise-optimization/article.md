# Roofline-guided stage-wise optimization

Also called: Roofline-guided optimization

## Overview
Roofline-guided stage-wise optimization is a methodology that uses stage-wise performance characterization to match optimization strategies to the dominant hardware constraints of the baseline pipeline, by reducing intermediate data movement in the front end and reducing arithmetic overhead in rasterization [S1, unofficial]. It solves the problem of uncoordinated software optimization in volume rendering contexts, where different stages of the rendering pipeline are bottlenecked by different hardware resources (memory bandwidth vs. compute throughput) [S1, unofficial]. It sets itself apart from generic optimization by explicitly using the Roofline model to identify the "knee" point and applying memory-centric optimizations to stages below it and compute-centric optimizations to stages near it [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The baseline 3D Gaussian Splatting (3DGS) rendering pipeline stages (preprocessing, sorting, rasterization), hardware specifications (peak compute throughput $P_{\text{peak}}$, memory bandwidth $BW$), and kernel execution traces.
*   **Output:** An optimized pipeline where specific stages are modified to address their identified bottlenecks.

**Data Structures:**
*   **Sorting Key:** A dynamic 32-bit integer packing tile index and quantized depth [S1, unofficial].
*   **Attributes:** Compressed Spherical Harmonics (SH) and geometric attributes [S1, unofficial].

**Method Steps:**
1.  **Roofline Characterization:** For each stage in the rendering pipeline, calculate the arithmetic intensity $I$ and the achievable performance $P$ using the Roofline model [S1, unofficial].
    $$P = \min(P_{\text{peak}}, BW \cdot I) \quad (1)$$
    Where:
    *   $P_{\text{peak}}$ is the peak compute throughput of the hardware [S1, unofficial].
    *   $BW$ is the memory bandwidth of the hardware [S1, unofficial].
    *   $I$ is the arithmetic intensity, defined as operations divided by data movement ($I = F/B$) [S1, unofficial].

2.  **Identify Bottleneck Regime:** Compare the stage's arithmetic intensity $I$ to the hardware's balance point (knee) $I_{\text{knee}}$ [S1, unofficial].
    $$I_{\text{knee}} = \frac{P_{\text{peak}}}{BW} \quad (2)$$
    *   If $I \ll I_{\text{knee}}$, the stage is bandwidth-bound (memory traffic sensitive) [S1, unofficial].
    *   If $I \approx I_{\text{knee}}$, the stage is compute-bound (arithmetic overhead sensitive) [S1, unofficial].

3.  **Apply Front-End Optimizations (Bandwidth-Limited Stages):** For stages identified as bandwidth-bound (typically preprocessing and sorting):
    *   Compress SH and geometric attributes to reduce memory footprint [S1, unofficial].
    *   Pack tile index and quantized depth into a single dynamic 32-bit sorting key to reduce intermediate data movement [S1, unofficial].
    *   Fuse preprocessing, tile duplication, and key generation into a single kernel to minimize global memory traffic [S1, unofficial].

4.  **Apply Rasterization Optimizations (Compute-Limited Stages):** For stages identified as compute-bound (typically rasterization):
    *   Evaluate two adjacent pixels simultaneously to increase instruction-level parallelism (ILP) [S1, unofficial].
    *   Replace standard exponential operations with a bit-level approximation tailored to alpha-thresholded blending to reduce arithmetic overhead [S1, unofficial].

**Parameters:**
*   **Hardware Balance Point ($I_{\text{knee}}$):** On an NVIDIA RTX 4090 GPU, $P_{\text{peak}} = 82.6 \text{ TFLOPS}$ and $BW = 1008 \text{ GB/s}$, resulting in $I_{\text{knee}} \approx 81.9 \text{ FLOPS/Byte}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The bit-level approximation in rasterization must be carefully calibrated to maintain visual fidelity while reducing arithmetic cost [S1, unofficial].
*   **Kernel Fusion:** Fusing too many stages may exceed register limits or warp occupancy thresholds, potentially degrading performance [G].
*   **Workload Scaling:** Optimizations tuned for a specific Gaussian count may not scale linearly; the total-frame speedup can be non-monotonic due to auxiliary overheads not eliminated by the primary optimization path [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def roofline_guided_optimization(pipeline_stages, hardware_specs):
    """
    Applies Roofline-guided optimizations to a rendering pipeline.
    
    Args:
        pipeline_stages: List of stage objects with 'name', 'flops', 'bytes' attributes.
        hardware_specs: Dict with 'P_peak' (TFLOPS) and 'BW' (GB/s).
        
    Returns:
        optimized_pipeline: List of optimized stage configurations.
    """
    P_peak = hardware_specs['P_peak']
    BW = hardware_specs['BW']
    I_knee = P_peak / BW
    
    optimized_stages = []
    
    for stage in pipeline_stages:
        # Calculate arithmetic intensity
        # I = F / B, where F is total FLOPS, B is total bytes transferred
        I_stage = stage.flops / stage.bytes
        
        if I_stage < I_knee:
            # Bandwidth-bound: Optimize for memory traffic reduction
            optimized_stage = optimize_bandwidth_bound_stage(stage)
        else:
            # Compute-bound: Optimize for arithmetic intensity/ILP
            optimized_stage = optimize_compute_bound_stage(stage)
            
        optimized_stages.append(optimized_stage)
        
    return optimized_stages

def optimize_bandwidth_bound_stage(stage):
    """
    Applies memory-centric optimizations.
    """
    if stage.name == "preprocessing":
        # Compress SH and geometric attributes
        stage.attributes = compress_attributes(stage.attributes)
        
        # Fuse preprocessing, tile duplication, and key generation
        stage.kernel = fuse_kernels([stage.preprocessing_kernel, 
                                     stage.tile_duplication_kernel, 
                                     stage.key_generation_kernel])
        
        # Pack tile index and quantized depth into 32-bit key
        stage.sort_key = pack_tile_index_and_depth(stage.tile_index, stage.quantized_depth)
        
    return stage

def optimize_compute_bound_stage(stage):
    """
    Applies compute-centric optimizations.
    """
    if stage.name == "rasterization":
        # Evaluate two adjacent pixels for ILP
        stage.pixel_processing = process_two_adjacent_pixels
        
        # Replace exp with bit-level approximation for alpha blending
        stage.blending_op = bit_level_alpha_approximation
        
    return stage
```

## Subtopics

### Synthetic Scale-Up Stress Diagnostic
A diagnostic method to isolate how sort/reordering and raster stages behave as Gaussian workload increases, using a fixed high-workload view from the bicycle scene [S2, unofficial]. It synthetically scales the workload from 5M to 30M Gaussians and measures median-of-5 timings under a locked RTX 4090 timing-only protocol [S2, unofficial]. At 30M emitted entries, the sort stage improves by 1.71x and the raster stage by 1.42x compared to the baseline [S2, unofficial]. At 5M points, Base Sort is 1.051ms, TileGS Sort is 0.387ms (2.72x speedup); Base Raster is 2.277ms, TileGS Raster is 1.554ms (1.46x speedup) [S2, unofficial]. At 30M points, Base Sort is 5.709ms, TileGS Sort is 3.340ms (1.71x speedup); Base Raster is 7.980ms, TileGS Raster is 5.603ms (1.42x speedup) [S2, unofficial]. The total-frame speedup is smaller and not monotonic because the diagnostic includes non-raster and auxiliary overheads not eliminated by the binned raster path [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The complexity depends on the specific stage optimizations. Sorting is typically $O(N \log N)$ or $O(N)$ with radix sort, where $N$ is the number of Gaussians [S1, unofficial]. Rasterization is $O(N \cdot T)$, where $T$ is the number of tiles [S1, unofficial].
*   Memory usage is reduced in the front end due to attribute compression and key packing [S1, unofficial].

**Measured Performance:**
*   On an NVIDIA RTX 4090 GPU, the FP32-DRAM balance point is approximately 81.9 FLOPS/Byte [S1, unofficial].
*   In the Synthetic Scale-Up Stress Diagnostic, TileGS (the implementation of these optimizations) showed consistent improvements in sort and raster stages as workload increased [S2, unofficial].

**Scaling:**
*   The absolute timing gaps between optimized and baseline stages widen with scale [S2, unofficial].
*   The total-frame speedup is not monotonic and is smaller than individual stage speedups due to auxiliary overheads [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Tailors optimizations to specific hardware bottlenecks, leading to significant speedups in critical stages (sort, raster) [S1, unofficial].
*   **Limitations:** The total-frame speedup may be limited by non-optimized auxiliary overheads [S2, unofficial]. The method requires detailed hardware-specific knowledge (e.g., $P_{\text{peak}}$, $BW$) [S1, unofficial].

**When to use it:**
*   When optimizing a rendering pipeline for a specific hardware target where different stages have distinct bottlenecks [S1, unofficial].
*   When memory bandwidth is a significant constraint in the front end [S1, unofficial].

**When not to use it:**
*   When the pipeline is uniformly compute-bound or memory-bound across all stages, as the stage-wise differentiation offers less benefit [G].
*   When hardware specifications are unknown or highly variable [G].

## Background
Roofline-guided stage-wise optimization is part of the RoofGS framework, presented in "RoofGS: Roofline-Guided End-to-End Acceleration of 3D Gaussian Splatting" [S1, unofficial]. It builds on the Roofline model [42, cited in S1] to identify dominant bottlenecks in the 3DGS rendering pipeline. The method addresses the limited attention to coordinated software optimization across the whole rendering pipeline in prior work [S1, unofficial]. It is used in the context of 3D Gaussian Splatting rendering acceleration [S1, unofficial].

## Sources
[S1] RoofGS: Roofline-Guided End-to-End Acceleration of 3D Gaussian Splatting, https://arxiv.org/abs/2608.15785, fetched 2026-10-29 (unofficial)
[S2] TileGS: Tile-Local Depth Binning for Gaussian Splatting Rasterization, https://doi.org/10.2312/pg.20261013, fetched 2026-10-29 (unofficial)
