# Sample Matching for Extinction Gradients

Also called: Sample Matching for Joint Extinction Gradient Estimation

## Overview
Sample Matching for Extinction Gradients is a differentiable volumetric path tracing method that reduces the variance of extinction coefficient gradients by evaluating the positive scattering term and the negative transmittance term at shared spatial locations along each path segment [S1, S2, unofficial]. Unlike the base [Volumetric Path Tracing](1863-volumetric-path-tracing.md) method, which estimates these two opposing terms at unrelated points, this technique exploits their negative correlation to cancel variance, while maintaining the same primal rendering output [S1, S2, unofficial].

## How it works
The method operates in two distinct phases: a primal pass identical to the base [Volumetric Path Tracing](1863-volumetric-path-tracing.md) integrator, and an adjoint pass that computes gradients via a two-kernel pipeline [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Scene geometry, participating media (defined by extinction $\sigma_t$, scattering $\sigma_s$, and emission $\sigma_e$), camera parameters, and the primal path records generated in the first pass [S1, S2, unofficial].
*   **Outputs:** Radiance image (identical to base method) and gradients of radiance with respect to the extinction coefficient $\sigma_t$ [S1, S2, unofficial].

### Data Structures
The core data structure is the **Segment Record**, written during the primal pass and read during the adjoint pass [S2, unofficial].

**Segment Record Layout:**
*   **Masked Scatter Values:** Fields generated from the variant's spectrum size (RGB or spectral) [S2, unofficial].
*   **Wavelengths:** The path's specific wavelengths, ensuring the second kernel evaluates the medium at the correct spectral channels [S2, unofficial].
*   **Segment Data:** Position, direction, and optical depth information for the specific segment [S2, unofficial].

### Parameters
*   **`segment_slots`:** The number of records a path may keep.
    *   **Default:** `max_depth + 1` (registered as `prbvolpath_sm_quad`). Every completed segment gets a record slot [S1, S2, unofficial].
    *   **Value 1:** One weighted reservoir per lane. The retained segment stands in for the whole path, scaled by the inverse selection probability (registered as `prbvolpath_sm_linear`) [S2, unofficial].
    *   **Value >1:** Linear mapping where segment $j$ goes to slot $\min(j, \text{segment\_slots} - 1)$. The first `segment_slots - 1` segments are kept exactly; the rest are pooled into one reserved row per lane via weighted reservoir sampling [S2, unofficial].
*   **`gradient_samples_per_segment`:** Number of gradient samples taken per segment.
    *   **Typical Values:** 1 or 4.
    *   **Trade-off:** Raising from 1 to 4 costs ~35% time but reduces variance [S2, unofficial].

### Algorithm Steps

#### 1. Primal Pass (Path Tracing)
The primal pass is identical to the base [Volumetric Path Tracing](1863-volumetric-path-tracing.md) method [S1, unofficial]. It performs no gradient sampling [S2, unofficial].

1.  Initialize path state.
2.  Loop through path depth:
    *   Sample interaction distance using [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) or [Delta tracking](985-delta-tracking.md) [S1, S2, unofficial].
    *   If a null collision occurs (empty space), skip efficiently [S2, unofficial].
    *   At interaction point $\mathbf{p}$:
        *   Evaluate scattering and emission terms.
        *   Update radiance contribution.
        *   **Write Segment Record:** Store masked scatter values and wavelengths for the current segment.
        *   **Reservoir Logic (if `segment_slots` < `max_depth + 1`):** Update the weighted reservoir running weight. Only this scalar is kept in loop state to avoid continuation spills [S2, unofficial].
    *   Terminate path via Russian Roulette or max depth.

#### 2. Adjoint Pass (Gradient Estimation)
The adjoint pass executes as two distinct kernels to avoid GPU continuation spills associated with tracing rays inside a loop [S2, unofficial].

**Kernel 1: Path Replay**
1.  Replay the primal path for each pixel.
2.  For each segment, write the corresponding **Segment Record** to a global buffer.
3.  The values in the record "die" at the end of the iteration; no gradient sampling occurs here [S2, unofficial].

**Kernel 2: Gradient Sampling**
1.  Read Segment Records from the buffer.
2.  For each record (one lane per record):
    *   Retrieve segment data (position $\mathbf{p}$, direction $\boldsymbol{\omega}$, wavelengths).
    *   **Sample Matching:**
        *   Trace fresh sub-paths from $\mathbf{p}$ to estimate the **scattering term** (positive contribution) [S2, unofficial].
        *   Trace fresh sub-paths from $\mathbf{p}$ to estimate the **transmittance term** (negative contribution) [S2, unofficial].
        *   *Note:* Reconnection (reusing the path's own continuation) was tested but dropped because it resulted in higher variance due to unstable transmittance ratios in the shift mapping weight [S2, unofficial].
    *   Compute the gradient estimate by combining the two terms evaluated at the same location [S1, S2, unofficial].
    *   Accumulate gradient into the output buffer.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Primal Pass (Identical to prbvolpath)
def primal_pass(scene, camera, rng):
    for pixel in camera.rays:
        path_radiance = 0.0
        throughput = 1.0
        current_pos = pixel.origin
        current_dir = pixel.direction
        
        # Reservoir state for segment selection (if needed)
        reservoir_weight = 0.0
        selected_segment_idx = -1
        
        for depth in range(max_depth):
            # Sample distance to next interaction
            dist = sample_distance(current_pos, current_dir, scene.medium)
            next_pos = current_pos + dist * current_dir
            
            # Evaluate medium at next_pos
            sigma_t = scene.medium.extinction(next_pos)
            sigma_s = scene.medium.scattering(next_pos)
            sigma_e = scene.medium.emission(next_pos)
            
            # Transmittance to this point
            tau = sigma_t * dist
            T = exp(-tau)
            
            # Update radiance
            path_radiance += throughput * sigma_e * T * dist # Simplified emission term
            
            # Sample scattering direction
            if sigma_s > 0:
                new_dir = sample_phase(current_dir, scene.phase)
                throughput *= (sigma_s / sigma_t) * T * scene.phase(current_dir, new_dir)
                
                # Write Segment Record
                record = {
                    'pos': next_pos,
                    'dir': current_dir,
                    'throughput': throughput,
                    'wavelengths': pixel.wavelengths,
                    'depth': depth
                }
                write_segment_record(record)
                
                # Reservoir Logic (if segment_slots < max_depth + 1)
                if segment_slots < max_depth + 1:
                    # Update reservoir weight
                    current_weight = throughput * sigma_s / sigma_t # Example weight
                    reservoir_weight += current_weight
                    if rng.uniform() < current_weight / reservoir_weight:
                        selected_segment_idx = depth
                        selected_segment_record = record
                
                current_pos = next_pos
                current_dir = new_dir
            else:
                break # No scattering, path ends or continues as pure transmission
                
        # If using reservoir, write the selected segment record explicitly
        if segment_slots == 1 and selected_segment_idx != -1:
            write_segment_record(selected_segment_record)

# Adjoint Pass Kernel 1: Replay
def adjoint_kernel_1(scene, camera, rng):
    for pixel in camera.rays:
        # Replay primal path logic exactly
        # Write segment records to global buffer
        pass

# Adjoint Pass Kernel 2: Gradient Sampling
def adjoint_kernel_2(scene, segment_records, rng):
    for record in segment_records:
        pos = record['pos']
        dir = record['dir']
        throughput = record['throughput']
        wavelengths = record['wavelengths']
        
        # Sample Matching: Evaluate both terms at 'pos'
        # 1. Scattering Term (Positive)
        scattering_grad = 0.0
        for _ in range(gradient_samples_per_segment):
            # Trace fresh sub-path from pos to estimate scattering contribution
            sub_path_radiance = trace_sub_path(pos, dir, scene, rng)
            scattering_grad += sub_path_radiance
        scattering_grad /= gradient_samples_per_segment
        
        # 2. Transmittance Term (Negative)
        transmittance_grad = 0.0
        for _ in range(gradient_samples_per_segment):
            # Trace fresh sub-path from pos to estimate transmittance sensitivity
            sub_path_transmittance = trace_sub_path_transmittance(pos, dir, scene, rng)
            transmittance_grad += sub_path_transmittance
        transmittance_grad /= gradient_samples_per_segment
        
        # Combine terms
        # The gradient is the difference between the scattering sensitivity and transmittance sensitivity
        # evaluated at the same point, leveraging negative correlation
        gradient = throughput * (scattering_grad - transmittance_grad)
        
        accumulate_gradient(gradient)
```

### Edge Cases and Pitfalls
*   **Continuation Spills:** Keeping a segment's record in loop state (using `dr.alloc_local` or named variables) causes +2.5 KiB/thread of continuation spills and +45% adjoint time on complex scenes (e.g., Bunny cloud) [S2, unofficial]. This is avoided by splitting the pass into two kernels [S2, unofficial].
*   **Reconnection Variance:** Reusing the path's own continuation for gradient samples (reconnection) results in higher variance than tracing fresh sub-paths. The shift mapping weight carries a transmittance ratio ranging from $10^{-9}$ to 356, and only 4.8% of samples could be reused [S2, unofficial].
*   **Unsupported Variants:** Polarized variants and forward mode are not supported, consistent with the base `prbvolpath` method [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The primal pass is identical to [Volumetric Path Tracing](1863-volumetric-path-tracing.md). The adjoint pass adds overhead proportional to `gradient_samples_per_segment` and the number of segments [S2, unofficial].
*   **Memory Complexity:** Depends on `segment_slots`.
    *   `segment_slots = 1`: ~32 MiB overhead (vs base) [S2, unofficial].
    *   `segment_slots = max_depth + 1`: ~4.6 GB overhead (vs base) on a typical wavefront [S2, unofficial].
*   **Performance:**
    *   **Primal Speedup:** The plugin can be faster than base `prbvolpath` in primal mode (e.g., 72.1 s vs 104.5 s) because it consumes null collisions in one fused C++ walk instead of paying `optixTrace` for each [S2, unofficial].
    *   **Gradient Variance:** On a Bunny cloud scene (768x576, 8 spp), sample matching with 4 slots reduced variance from 136.5 (DRT reference) to 46.4 (occupied voxels) and 81.4 (all voxels), a 0.26x factor [S2, unofficial].
    *   **Accuracy:** Gradients agree with finite differences within 1% in RGB and spectral variants [S1, unofficial].
*   **Strengths:** Significant variance reduction for extinction gradients; supports spectral and RGB variants; efficient primal rendering.
*   **Limitations:** High memory usage with default `segment_slots`; adjoint pass is slower than primal; not suitable for polarized or forward-mode differentiation [S1, S2, unofficial].

## Background
*   **Origin:** Introduced in 2026 as part of the Mitsuba 3 renderer development [S1, S2, unofficial].
*   **Development:** The method was developed to address the high variance of extinction gradients in differentiable volumetric path tracing. Initial attempts using reconnection (reusing path continuations) were abandoned due to instability in transmittance ratios [S2, unofficial].
*   **Usage:** Implemented as the `prbvolpath_sm` plugin in Mitsuba 3, supporting both RGB and spectral variants [S2, unofficial].

## Sources
[S1] Add prbvolpath_sm: new unbiased differentiable volume integrator with sample matching, https://github.com/mitsuba-renderer/mitsuba3/pull/1939, fetched 2026-10-26 (unofficial)
[S2] Sample matching for extinction gradients (review branch), https://github.com/AuroraRyan0301/mitsuba3/pull/1, fetched 2026-10-26 (unofficial)
