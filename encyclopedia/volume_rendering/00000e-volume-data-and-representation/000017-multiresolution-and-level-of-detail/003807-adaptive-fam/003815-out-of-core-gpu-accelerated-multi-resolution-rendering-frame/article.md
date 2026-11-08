# Out-of-Core GPU-Accelerated Multi-Resolution Rendering Framework

Also called: Out-of-core multi-resolution rendering framework, Adaptive-FAM rendering pipeline

## Overview
This framework provides an out-of-core rendering pipeline for large-scale volumetric datasets, managing data movement between storage, system RAM, and GPU VRAM to maintain interactive frame rates [S1, unofficial]. It builds on [Adaptive-FAM](8156-adaptive-fam.md) by utilizing its micro-models to enable multi-resolution visualization, where the level of detail is selected based on the viewer's distance to the data [S1, unofficial]. Unlike in-core methods, it explicitly handles the latency of disk-to-RAM transfers through a dedicated caching and prefetching subsystem [S1, unofficial].

## How it works
The framework consists of three parallel procedures: Caching, Prefetching, and Rendering. The rendering procedure executes on the GPU, while Caching and Prefetching manage data loading on the CPU [S1, unofficial].

### Inputs and Outputs
*   **Input:** A sequence of Points of View (POVs) $POV_n$; a large-scale volumetric dataset stored on disk; Adaptive-FAM encoded micro-models.
*   **Output:** A rendered 2D image for each POV.
*   **Data Types:** All volumetric data uses `float32` [S1, unofficial]. Transfer functions map scalar values to Color and Opacity [S1, unofficial].

### Data Structures
*   **Micro-model Cache:** A fixed-size LRU (Least Recently Used) cache stored in system RAM. The experimental setup fixes this size at 200 micro-blocks [S1, unofficial].
*   **LOD Selection:** A bipartite strategy partitions micro-blocks into four Levels of Detail (LODs) based on the distance $d$ from the POV to the micro-block centroid [S1, unofficial]. The distance ranges are:
    *   LOD 0: $[0, 0.8]$
    *   LOD 1: $[0.8, 1.6]$
    *   LOD 2: $[1.6, 2.4]$
    *   LOD 3: $[2.4, \infty]$
    [S1, unofficial]

### Rendering Procedure (GPU)
The rendering uses direct volume rendering (DVR) with ray casting. It queries values and gradients directly from the Adaptive-FAM micro-models resident in GPU VRAM [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render(visible_micro_models, row_start, row_end, col_start, col_end):
    rendering_done = False
    for i from row_start to row_end:
        for j from col_start to col_end:
            color = vec4(0.0, 0.0, 0.0, 0.0)
            O_cumulative = 0.0
            O_max = 1.0 # Threshold for early termination
            
            for s from SampleFirst to SampleLast:
                x, y, z = get_sample_location(s)
                
                # Query Adaptive-FAM micro-model
                v = value_micro_model(x, y, z)
                gx, gy, gz = gradient_micro_model(x, y, z)
                
                # Apply Transfer Function
                apply_tf(v, color) # Maps v to color.rgb and alpha
                
                # Shading
                c_diffuse, c_specular = light_coefficient(gx, gy, gz)
                add_shading(c_diffuse, c_specular, color)
                
                # Front-to-back compositing
                O_cumulative = do_color_compositing(color, O_cumulative)
                
                # Early termination
                if O_cumulative > O_max:
                    break
            
            set_pixel_color(i, j, color)
    rendering_done = True
```

### Caching Procedure (CPU)
Caching utilizes spatial locality to reduce data movement frequency [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cache_update(current_pov, cache, storage):
    # 1. Find visible micro-models
    vec_vm = find_visible_micro_models(current_pov)
    
    # 2. Update cache in system RAM
    for m in vec_vm:
        if m in cache:
            continue # Cache hit
        else: # Cache miss
            if cache.is_not_full():
                cache.load(m, storage)
            else:
                lru_micro = cache.get_lru()
                cache.replace(lru_micro, m, storage)
    
    # 3. Copy to GPU VRAM
    cache.copy_to_gpu()
```

### Prefetching Procedure (CPU)
Prefetching predicts visible micro-models for the next POV to hide storage latency. It runs on a separate CPU thread and is preempted when rendering finishes [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prefetch_next_pov(predictor, cache, storage, rendering_done_flag):
    while not rendering_done_flag:
        # 1. Predict next POV
        next_pov = predictor.predict()
        
        # 2. Predict visible micro-models for next POV
        vec_vm_next = find_visible_micro_models(next_pov)
        
        # 3. Load missing micro-models into system cache
        for m in vec_vm_next:
            if m not in cache:
                if cache.is_not_full():
                    cache.load(m, storage)
                else:
                    lru_micro = cache.get_lru()
                    cache.replace(lru_micro, m, storage)
        
        # Note: Preemption occurs if rendering_done_flag becomes true
```

### Parameters
*   **Cache Size:** 200 micro-blocks [S1, unofficial].
*   **LOD Ranges:** $[0, 0.8], [0.8, 1.6], [1.6, 2.4], [2.4, \infty]$ [S1, unofficial].
*   **Prefetching Algorithms:** APPA (predicts parent POV), ForeCache (predicts next POV based on user behavior), or LSTM (predicts next POV based on historical data) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Prefetch Preemption:** The prefetching thread must check for rendering completion to avoid wasting I/O bandwidth on data that will not be used [S1, unofficial].
*   **Storage Latency:** The framework is designed to handle high-latency storage (e.g., 5400 RPM HDD). Performance degrades if the cache hit rate is low due to non-local camera movements [S1, unofficial].
*   **VRAM Capacity:** The GPU VRAM must hold the visible micro-models. If the number of visible micro-blocks exceeds VRAM capacity, rendering will fail or stall [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ where $N$ is the number of pixels and samples. Caching and Prefetching are $O(M)$ where $M$ is the number of visible micro-blocks, dominated by I/O latency.
*   **Memory Complexity:** System RAM usage is fixed by the cache size (200 micro-blocks in experiments). GPU VRAM usage depends on the number of visible micro-blocks [S1, unofficial].
*   **Performance:** Tested on an NVIDIA GTX 1080 Ti (8GB VRAM) with an Intel Core i7-7700K CPU and 16GB DDR4 RAM [S1, unofficial]. The method demonstrates effectiveness in interactive visualization of large-scale datasets by mitigating the bottleneck of data movement from storage to RAM [S1, unofficial].
*   **Strengths:** Handles datasets larger than VRAM; adaptive encoding reduces data volume; multi-threading hides I/O latency.
*   **Limitations:** Performance depends heavily on the accuracy of the prefetching algorithm; requires significant CPU resources for I/O management; fixed cache size may not be optimal for all scene complexities [S1, unofficial].

## Background
The framework was introduced in the paper "Adaptive Multi-Resolution Encoding for Interactive Large-Scale Volume Visualization through Functional Approximation" [S1, unofficial]. It extends the Adaptive-FAM encoding method, which uses micro-blocks and functional approximation to compress volumetric data [S1, unofficial]. The work addresses the challenge of visualizing large-scale volumes (e.g., micro-CT scans, fluid simulations) that exceed GPU memory capacity [S1, unofficial].

## Sources
[S1] Adaptive Multi-Resolution Encoding for Interactive Large-Scale Volume Visualization through Functional Approximation, https://www.researchgate.net/publication/383700795_Adaptive_Multi-Resolution_Encoding_for_Interactive_Large-Scale_Volume_Visualization_through_Functional_Approximation, fetched 2026-10-30 (unofficial)
