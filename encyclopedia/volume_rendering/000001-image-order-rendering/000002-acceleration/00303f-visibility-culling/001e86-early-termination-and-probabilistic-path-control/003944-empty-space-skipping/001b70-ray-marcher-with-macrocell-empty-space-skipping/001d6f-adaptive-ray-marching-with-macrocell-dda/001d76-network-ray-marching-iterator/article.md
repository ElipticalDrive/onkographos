# Network Ray Marching Iterator

Also called: Distributed Ray Marching with Macrocell DDA, Networked Ray Marching

## Overview
The Network Ray Marching Iterator is a ray marching algorithm designed for distributed or networked rendering environments, typically leveraging GPU ray tracing APIs such as NVIDIA OptiX [S1, unofficial]. It extends the parent [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md) by introducing block-level divergence control via `block_any` to manage thread synchronization across macrocell boundaries [S1, unofficial]. This method is distinct from its siblings because it explicitly tracks the "alive" status of rays in a distributed context, ensuring that inactive threads do not waste computation while maintaining adaptive sampling rates based on opacity upper bounds [S1, unofficial].

## How it works
The algorithm traverses a volume using a 3D Digital Differential Analyzer (DDA) over macrocells. It adapts the step size within each macrocell based on an estimated opacity upper bound and uses a device function `block_any` to synchronize thread divergence across the block [S1, unofficial].

### Inputs and Outputs
- **Inputs**: Ray origin `org`, ray direction `dir`, ray segment `[tMin, tMax]`, volume dimensions `dims`, macrocell spacing inverse `macrocell_spacings_rcp`, and a lambda function defining the per-sample body logic [S1, unofficial].
- **Outputs**: The final "alive" status of the ray after traversal, and accumulated color/opacity if the body function modifies state [S1, unofficial].

### Data Structures
- **Macrocell Coordinates**: `m_org` and `m_dir` are floating-point vectors representing the ray origin and direction scaled by the inverse macrocell spacing [S1, unofficial].
- **DDA State**: The DDA yields a tuple `(cell, t0, t1, alive)` for each macrocell intersection, where `cell` is the macrocell index, `t0` and `t1` are the entry and exit distances, and `alive` is the boolean status of the ray [S1, unofficial].

### Parameters
- **`block_any`**: A device function that returns `true` if any thread in the current block satisfies a given condition [S1, unofficial].
- **`float_epsilon`**: A small constant used to determine if the opacity upper bound is effectively zero. Value not given in the sources [G].
- **`step`**: The base sampling step size. Value not given in the sources [G].

### Steps
1. **Calculate Macrocell Coordinates**: Compute the ray origin and direction in macrocell space: $m\_org = org \times macrocell\_spacings\_rcp$, $m\_dir = dir \times macrocell\_spacings\_rcp$ [S1, unofficial].
2. **Invoke DDA**: Call the DDA traversal with `m_org`, `m_dir`, `tMin`, `tMax`, `dims`, and a lambda callback [S1, unofficial].
3. **Initialize Loop Variables**: Inside the lambda, initialize `ss = 100.f` (adaptive step size) and `non_empty = true` [S1, unofficial].
4. **Check Opacity Upper Bound**: If `alive` is true, calculate `r = opacityUpperBound(self, cell)` [S1, unofficial]. If `fabsf(r) <= float_epsilon`, set `non_empty = false` [S1, unofficial].
5. **Calculate Adaptive Step**: If `alive` is true, set `ss = adaptiveSamplingRate(step, r)` [S1, unofficial].
6. **Divergence Control**: Evaluate `block_any(non_empty && alive)` [S1, unofficial]. If this returns `false`, return `alive` (terminate the ray for this block) [S1, unofficial].
7. **Initialize Sample Interval**: Set `t = vec2f(t0, min(t1, t0 + ss))` [S1, unofficial].
8. **Sample Loop**: Loop while `block_any(alive && (t.y > t.x))`: update `alive = body(t, alive)`, `t.x = t.y`, `t.y = min(t.x + ss, t1)` [S1, unofficial].
9. **Return Status**: Return the final `alive` status [S1, unofficial].

### Non-Adaptive Fallback
If adaptive sampling is not used, initialize `t = vec2f(tMin, min(tMax, tMin + step))` and `alive = (t.y > t.x)` [S1, unofficial]. Loop while `block_any(alive && (t.y > t.x))`: update `alive = body(t, alive)`, `t.x = t.y`, `t.y = min(t.x + step, tMax)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_any(condition_list):
    """Device function: returns True if any element in the list is True."""
    return any(condition_list)

def network_ray_marching_iterator(org, dir, t_min, t_max, dims, macrocell_spacings_rcp, step, float_epsilon, opacity_upper_bound, adaptive_sampling_rate, body):
    # 1. Calculate macrocell coordinates
    m_org = [org[i] * macrocell_spacings_rcp for i in range(3)]
    m_dir = [dir[i] * macrocell_spacings_rcp for i in range(3)]

    # 2. Call DDA
    def lambda_func(cell, t0, t1, alive):
        # 3. Initialize loop variables
        ss = 100.0
        non_empty = True

        # 4. Check opacity upper bound
        if alive:
            r = opacity_upper_bound(cell)
            if abs(r) <= float_epsilon:
                non_empty = False

        # 5. Calculate adaptive step
        if alive:
            ss = adaptive_sampling_rate(step, r)

        # 6. Divergence control
        if not block_any([non_empty and alive]):
            return False

        # 7. Initialize sample interval
        t_x = t0
        t_y = min(t1, t0 + ss)

        # 8. Sample loop
        while block_any([alive and (t_y > t_x)]):
            alive = body(t_x, t_y, alive)
            t_x = t_y
            t_y = min(t_x + ss, t1)

        # 9. Return status
        return True

    # Standard 3D DDA implementation (replaces network_dda3)
    # Fixes initialization bug where cell/t_max_edge were incorrectly sized/initialized
    cell = [int(m_org[i]) if m_org[i] >= 0 else int(m_org[i]) - 1 for i in range(3)]
    cell = [max(0, min(dims[i] - 1, cell[i])) for i in range(3)]

    step_dir = [1 if m_dir[i] > 0 else -1 for i in range(3)]
    t_delta = [1.0 / abs(m_dir[i]) if m_dir[i] != 0 else float('inf') for i in range(3)]

    t_max_edge = [
        (cell[i] + 1.0 - m_org[i]) if m_dir[i] > 0 else (cell[i] - m_org[i])
        for i in range(3)
    ]

    t0 = max(t_min, min(t_max_edge))

    while t0 < t_max:
        min_idx = 0
        min_val = t_max_edge[0]
        for i in range(1, 3):
            if t_max_edge[i] < min_val:
                min_val = t_max_edge[i]
                min_idx = i

        t1 = min(min_val, t_max)
        alive = True

        if not lambda_func(cell, t0, t1, alive):
            return False

        t_max_edge[min_idx] += t_delta[min_idx]
        cell[min_idx] += step_dir[min_idx]
        cell[min_idx] = max(0, min(dims[min_idx] - 1, cell[min_idx]))
        t0 = t1

    return False
```

### Edge Cases and Pitfalls
- **Precision**: The use of `float_epsilon` for opacity checks determines when a macrocell is considered empty [S1, unofficial].
- **Boundaries**: The `min(t1, t0 + ss)` logic ensures that the last sample does not exceed the macrocell exit distance `t1` [S1, unofficial].
- **Zero Direction Components**: The DDA handles rays parallel to grid axes by using infinite deltas for those dimensions [G].

## Subtopics

### [Resumable Ray Marching Iterator](1505-resumable-ray-marching-iterator.md)
A variant that allows ray traversal to be paused and resumed, useful for progressive rendering. Choose when memory bandwidth is limited or when integrating with progressive refinement techniques.

### [Macrocell Grid Traversal with Majorant Densities](1528-macrocell-grid-traversal-with-majorant-densities.md)
Uses majorant densities for more accurate opacity bounds. Choose when higher accuracy in empty-space skipping is required, at the cost of increased computation per macrocell.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Optimized for distributed rendering on GPUs using ray tracing APIs like OptiX [S1, unofficial].
- **Strengths**: Efficient handling of thread divergence in distributed contexts; adaptive sampling reduces unnecessary computations [S1, unofficial].
- **Limitations**: Requires support for `block_any` and custom DDA implementations; may not be suitable for non-GPU or non-ray-tracing environments [S1, unofficial].

## Background
The Network Ray Marching Iterator was introduced as part of the instantvnr project, a neural volume rendering framework [S1, unofficial]. It builds on the principles of adaptive ray marching and macrocell traversal, extending them to support distributed rendering scenarios [S1, unofficial]. The method is designed to leverage the parallelism of modern GPUs and the synchronization primitives provided by ray tracing APIs [S1, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
