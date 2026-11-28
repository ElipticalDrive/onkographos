# Unidirectional Spectral Tracking


## Overview
Unidirectional Spectral Tracking is a spectral volume rendering technique that samples absorption, scattering, and null events using a majorant density to drive the sampling process, returning the path length and spectral contribution while handling Russian roulette implicitly via event selection [S1, unofficial].

## How it works
**Inputs and Outputs:**
* **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a Voxelized Path Tracing Density field (`vsdf`); a maximum voxel density `max_density`; a maximum ray distance `max_distance`; a random number generator `rng` [S1, unofficial].
* **Output:** A pair containing the total `path_length` traveled and the spectral `throughput` (radiance contribution) [S1, unofficial].

**Data Structures:**
* `vsdf`: A data structure providing density evaluation. It returns a vector of total density and a scalar scatter ratio (albedo) at a given position [S1, unofficial].
* `rng`: A random number generator capable of producing uniform floating-point numbers in $[0, 1)$ [S1, unofficial].

**Parameters:**
* `max_density`: The maximum voxel density of the volume multiplied by an object density multiplier ($\rho_{max} = \text{voxel\_max} \times \text{density\_mult}$) [S1, unofficial].
* `imax_density`: The inverse of `max_density` ($1.0 / \text{max\_density}$) [S1, unofficial].
* `majorant_density`: The majorant used for distance sampling. In this implementation, it is the mean of the scatter product with `max_density`: $\bar{\sigma}_M = \text{mean}(\text{vsdf.scatter} \times \text{max\_density})$ [S1, unofficial].

**Algorithm Steps:**
1. Initialize `path_length` to $0.0$, `transmittance` $\mathbf{T}$ to $\vec{1}$, and `throughput` $\mathbf{f}$ to $\vec{1}$ [S1, unofficial].
2. Loop indefinitely:
   a. **Sample Path Length:** If `majorant_density` is $0.0$, set `path_length` to $-\text{FLT\_MAX}$ [S1, unofficial]. Otherwise, sample $u_1 \sim \mathcal{U}[0,1)$ and compute:
      $$ \text{path\_length} \leftarrow \text{path\_length} - \frac{\ln(1 - u_1)}{\text{majorant\_density}} $$
   b. **Termination:** If `path_length` $\ge$ `max_distance`, break the loop [S1, unofficial].
   c. **Position:** Calculate current position:
      $$ \mathbf{p} = \mathbf{o} + \text{path\_length} \times \mathbf{d} $$
   d. **Evaluate Cross-sections:** Evaluate the VPT density at $\mathbf{p}$ to get total density vector $\boldsymbol{\sigma}_t$ and scatter ratio $s$ [S1, unofficial]:
      $$ \boldsymbol{\sigma}_t = \text{eval\_vpt\_density}(\text{vsdf}, \mathbf{p}) $$
      $$ s = \text{vsdf.scatter} $$
      Compute component cross-sections:
      $$ \boldsymbol{\sigma}_s = \boldsymbol{\sigma}_t \times s $$
      $$ \boldsymbol{\sigma}_a = \boldsymbol{\sigma}_t \times (\vec{1} - s) $$
      $$ \boldsymbol{\sigma}_n = \text{max\_density} \times \vec{1} - \boldsymbol{\sigma}_t $$
   e. **Update Transmittance:**
      $$ \mathbf{T} \leftarrow \mathbf{T} \times (\vec{1} - \boldsymbol{\sigma}_t \times \text{imax\_density}) $$
   f. **Event Selection:** Determine the event type $e$ and the corresponding coefficient $\mu_e$ [S1, unofficial]. Sample $e \in \{\text{EVENT\_NULL}, \text{EVENT\_SCATTER}, \text{EVENT\_ABSORB}\}$ proportionally to the norms of $\boldsymbol{\sigma}_n$, $\boldsymbol{\sigma}_s$, and $\boldsymbol{\sigma}_a$ [G].
      * If $e = \text{EVENT\_NULL}$, $\mu_e = \boldsymbol{\sigma}_n$ [S1, unofficial].
      * If $e = \text{EVENT\_SCATTER}$, $\mu_e = s$ (scalar, applied to throughput) [S1, unofficial].
      * If $e = \text{EVENT\_ABSORB}$, $\mu_e = \vec{1} - s$ (scalar, applied to throughput) [S1, unofficial].
   g. **Null Event:** If $e = \text{EVENT\_NULL}$, continue to the next iteration (loop back to step 2a) [S1, unofficial].
   h. **Update Throughput:**
      $$ \mathbf{f} \leftarrow \mathbf{f} \times \mu_e $$
      $$ p = \mathbf{f} $$
   i. **Return:** Return the pair `(path_length, f / mean(p))` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def unidirectional_spectral_tracking(ray_o, ray_d, vsdf, max_density, max_distance, rng):
    imax_density = 1.0 / max_density
    # [G] majorant_density is precomputed as the mean of scatter product with max_density
    majorant_density = max_density * vsdf.scatter

    path_length = 0.0
    transmittance = [1.0, 1.0, 1.0]
    throughput = [1.0, 1.0, 1.0]

    while True:
        if majorant_density == 0.0:
            path_length = -float('inf')
        else:
            u1 = rng.uniform(0.0, 1.0)
            path_length -= math.log(1.0 - u1) / majorant_density

        if path_length >= max_distance:
            break

        current_pos = [ray_o[i] + path_length * ray_d[i] for i in range(3)]
        sigma_t = eval_vpt_density(vsdf, current_pos)
        scatter_ratio = vsdf.scatter

        sigma_s = [sigma_t[i] * scatter_ratio for i in range(3)]
        sigma_a = [sigma_t[i] * (1.0 - scatter_ratio) for i in range(3)]
        sigma_n = [max_density - sigma_t[i] for i in range(3)]

        transmittance = [transmittance[i] * (1.0 - sigma_t[i] * imax_density) for i in range(3)]

        # [G] Standard spectral event selection proportional to coefficient norms
        norm_n = math.sqrt(sum(x**2 for x in sigma_n))
        norm_s = math.sqrt(sum(x**2 for x in sigma_s))
        norm_a = math.sqrt(sum(x**2 for x in sigma_a))
        total_norm = norm_n + norm_s + norm_a

        if total_norm == 0:
            e = "EVENT_NULL"
        else:
            u2 = rng.uniform(0.0, 1.0)
            if u2 < norm_n / total_norm:
                e = "EVENT_NULL"
            elif u2 < (norm_n + norm_s) / total_norm:
                e = "EVENT_SCATTER"
            else:
                e = "EVENT_ABSORB"

        if e == "EVENT_NULL":
            mu_e = sigma_n
        elif e == "EVENT_SCATTER":
            mu_e = scatter_ratio
        else:
            mu_e = 1.0 - scatter_ratio

        if e == "EVENT_NULL":
            continue

        # Element-wise multiplication: if mu_e is scalar, broadcast; if vec, element-wise
        if isinstance(mu_e, (int, float)):
            throughput = [throughput[i] * mu_e for i in range(3)]
        else:
            throughput = [throughput[i] * mu_e[i] for i in range(3)]

        p = throughput[:]
        mean_p = sum(p) / 3.0
        if mean_p == 0.0:
            return path_length, [0.0, 0.0, 0.0]
        return path_length, [x / mean_p for x in throughput]

def eval_vpt_density(vsdf, pos):
    # [G] Standard VPT density lookup returning a vec3
    return vsdf.density_at(pos)
```

**Edge Cases and Pitfalls:**
* **Zero Majorant:** If `majorant_density` is $0.0$, the distance sampling formula involves division by zero. The implementation must handle this by setting `path_length` to $-\text{FLT\_MAX}$ (or a sufficiently large value) to effectively terminate or skip the step [S1, unofficial].

## Operating characteristics
Time and memory complexity: Not given in the sources [S1, unofficial].
Measured performance, scaling, strengths, limitations, and usage conditions: Not given in the sources [S1, unofficial].

## Background
Origin, history, development, and usage: Not given in the sources [S1, unofficial].

## Sources
[S1] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
