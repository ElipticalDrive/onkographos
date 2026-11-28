# Path Tracing with Sample Streaming

Also called: Streaming Path Tracing

## Overview
Path Tracing with Sample Streaming is a unidirectional path tracing algorithm designed for efficient rendering of participating media, particularly when using high-cost volume representations like neural fields. It solves the performance bottleneck of evaluating volume samples by decoupling the computation of sample coordinates from the shading logic, organizing data in a structure-of-arrays (SoA) format to facilitate coalesced global memory accesses [S1]. Unlike standard implementations that process rays sequentially or in large rigid batches, this method maintains a single active ray state that alternates between scattering and shadow modes, using Woodcock tracking for distance sampling and optimizing throughput via sample streaming [S1].

## How it works
The method implements a unidirectional path tracer where each ray alternates between a **scattering state** (propagating through the medium) and a **shadow state** (estimating transmittance to light sources). The core innovation is the separation of the `sample` function (which computes the next interaction coordinate) from the `shade` function (which handles throughput, Russian Roulette, and state transitions), allowing for batched, coalesced memory access patterns [S1].

### Inputs and Outputs
*   **Inputs**: A set of primary rays; a volume representation $\mathcal{V}$ capable of evaluating density $\sigma_t(\mathbf{x})$ and phase function $p(\mathbf{x}, \omega_i, \omega_o)$; light source positions and intensities.
*   **Outputs**: Accumulated pixel colors.

### Data Structures
The implementation uses a **Structure-of-Arrays (SoA)** layout for rays to maximize memory coalescing during GPU execution [S1]. Key fields per ray include:
*   `org`: Origin point.
*   `next_coordinate`: The result of the last Woodcock tracking step.
*   `throughput`: Accumulated radiance weight.
*   `shadow`: Boolean flag indicating if the ray is currently in the shadow state (estimating transmittance) or scattering state.
*   `valid`: Boolean indicating if the ray is still active within the volume.

### Woodcock Tracking
Distance sampling is performed using Woodcock tracking (delta tracking) [S1]. Given a maximum extinction coefficient $\mu_{\max}$, a sample coordinate $t$ is generated using uniform random variables $\zeta, \xi \sim U[0,1]$:

$$
t = -\frac{\ln(1-\zeta)}{\mu_{\max}}
$$

A collision is accepted (real collision) if:

$$
\xi < \frac{\sigma_t(\mathbf{x} + t\omega)}{\mu_{\max}}
$$

Otherwise, it is a null collision, and the process repeats with a new sample. If the ray exits the volume boundary without a real collision, it returns `INVALID` [S1].

### Algorithm Steps
1.  **Initialization**: Initialize primary rays. Set `shadow = false` (scattering state).
2.  **Sample Coordinate (`sample` function)**:
    *   If the ray has a pre-computed hit (`ray.has_hit(t)`), set `next_coordinate = ray.at(t)` and return `true` [S1].
    *   If the ray is in the shadow state (`ray.shadow` is true):
        *   Compute direct lighting contribution using the current coordinate.
        *   Set `shadow = false` to switch to scattering state.
        *   Re-generate a new sample coordinate using Woodcock tracking.
        *   If a hit is found, update `next_coordinate` and return `true` [S1].
    *   If neither condition is met, return `false` [S1].
3.  **Shade and Advance (`shade` function)**:
    *   If the ray has no valid collision (`is_null_collision()`), terminate the ray and return `true` (done) [S1].
    *   If the ray is in the shadow state:
        *   Set `shadow = false` (switch to scattering) [S1].
    *   Else (ray is in scattering state):
        *   Apply Russian Roulette. If terminated, return `false` [S1].
        *   Update throughput: `throughput *= phase_function(params, ray)` [S1].
        *   Advance ray origin: `org = next_coordinate` [S1].
        *   Set `shadow = true` (switch to shadow state) [S1].
    *   Perform volume intersection check (`intersect_volume()`) to determine if the ray remains within the bounding volume [S1].

### Pseudocode
The following pseudocode combines the logic from the sources into a single coherent flow. It assumes helper functions `woodcock_track(ray)` returns a hit distance or `INVALID`, and `russian_roulette(ray)` returns `true` if the ray is terminated.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_streaming(ray):
    """
    Computes the next sample coordinate.
    Handles conversion from shadow to scattering state.
    """
    # Check if a coordinate was already computed in a previous step
    if ray.has_hit():
        t = ray.get_hit_distance()
        ray.next_coordinate = ray.at(t)
        return True
    
    # If currently in shadow state, we need to switch to scattering
    if ray.shadow:
        # 1. Compute direct lighting / transmittance for the shadow ray
        #    This step uses the current ray.org and the light source
        compute_direct_lighting(ray)
        
        # 2. Switch state to scattering
        ray.shadow = False
        
        # 3. Re-generate a coordinate for the new scattering path
        #    This involves Woodcock tracking from the current position
        t = woodcock_track(ray)
        
        if ray.has_hit(t):
            ray.next_coordinate = ray.at(t)
            return True
            
    return False

def shade_streaming(params, ray):
    """
    Handles throughput, Russian Roulette, and state transitions.
    """
    # If no collision occurred in the last step, the ray is done
    if ray.is_null_collision():
        return True # Ray terminated successfully (exited volume)
    
    # If the ray is in shadow state, we just finished shading it.
    # We need to switch to scattering state for the next step.
    if ray.shadow:
        ray.shadow = False
        # Note: The actual coordinate generation happens in sample()
    else:
        # Ray is in scattering state.
        
        # 1. Russian Roulette
        if russian_roulette(ray):
            return False # Ray terminated by RR
            
        # 2. Update throughput with phase function
        ray.throughput *= phase_function(params, ray)
        
        # 3. Advance the ray origin to the interaction point
        ray.org = ray.next_coordinate
        
        # 4. Switch to shadow state for the next iteration
        ray.shadow = True
        
    # 5. Check if the ray is still inside the volume
    #    This prepares the ray for the next sample() call
    return ray.intersect_volume()

def render_kernel(params, num_rays):
    """
    Main loop for streaming path tracing.
    """
    for ray in rays:
        # Initialize ray (origin, direction, throughput=1, shadow=False)
        init_ray(ray)
        
        while ray.is_valid():
            # Step 1: Compute next coordinate
            # This may convert shadow->scattering and generate a hit
            has_hit = sample_streaming(ray)
            
            if has_hit:
                # Save ray state for next iteration or accumulation
                save_ray_state(params, ray)
            else:
                # No hit or termination
                accumulate_pixel_color(params, ray)
                break
                
            # Step 2: Shade and transition
            # This handles RR, throughput, and state switch
            done = shade_streaming(params, ray)
            if done:
                break
```

### Parameters and Edge Cases
*   **$\mu_{\max}$**: The maximum extinction coefficient in the volume. Must be $\ge \max(\sigma_t(\mathbf{x}))$. Too high leads to inefficient null collisions; too low causes invalid sampling [S1].
*   **Russian Roulette**: Used to terminate paths with low throughput. The probability of survival should balance variance and bias [S1].
*   **Null Collisions**: If Woodcock tracking returns a null collision, the loop continues. The implementation must handle `INVALID` returns correctly to avoid infinite loops [S1].
*   **Precision**: Floating-point precision in `woodcock_track` can affect boundary accuracy. Not given in the sources.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot L)$, where $N$ is the number of rays and $L$ is the average path length. The cost per step is dominated by volume evaluation $\sigma_t(\mathbf{x})$ and phase function evaluation.
*   **Memory Complexity**: $O(N)$ for the SoA ray buffer. The SoA layout reduces memory bandwidth pressure compared to Structure-of-Structures (SoS) [S1].
*   **Performance**: Optimized for neural volume representations where evaluation is expensive. By batching sample coordinate computation (`sample` kernel) separately from shading (`shade` kernel), it improves GPU occupancy and memory coalescing [S1].
*   **Strengths**: High performance on GPU for neural volumes; efficient memory access patterns; correct handling of volumetric scattering and shadowing in a unified loop.
*   **Limitations**: Requires careful tuning of $\mu_{\max}$; variance can be high in dense media; not inherently differentiable (unlike [Unidirectional Differentiable Path Tracing](7780-unidirectional-differentiable-path-tracing.md)) [S1].
*   **When to use**: Real-time or interactive visualization of neural radiance fields or other high-cost volume representations [S1].
*   **When not to use**: Simple homogeneous media where analytical solutions exist; scenes requiring differentiable rendering [S1].

## Background
The method was introduced in the context of interactive volume visualization using multi-resolution hash encoding, specifically to address the computational cost of neural volume representations [S1]. It builds on standard unidirectional path tracing principles but adapts the loop structure to support sample streaming and state alternation [S1]. The approach was verified against a reference mega-kernel implementation [S1].

## Sources
[S1] Interactive Volume Visualization via Multi-Resolution Hash Encoding, https://arxiv.org/abs/2207.11620, fetched 2026-10-26
