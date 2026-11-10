# Path Tracing with Discrete Event Sampling

Also called: Null-Event Path Tracing, Russian Roulette with Discrete Events

## Overview
Path Tracing with Discrete Event Sampling is a path tracing algorithm for participating media that replaces continuous Russian Roulette termination with a discrete probabilistic decision between absorption, scattering, and null events at each sampled point [S1, unofficial]. Unlike [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md), which uses continuous probability based on albedo/extinction ratios, this method explicitly handles null events (transmittance without interaction) to improve efficiency in sparse media by skipping non-interacting steps [S1, unofficial]. It supports temperature-dependent emission and Henyey-Greenstein phase function scattering [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, unofficial].
- Volume grid providing density $\sigma_t(\mathbf{x})$, temperature $T(\mathbf{x})$ (optional), and emission properties [S1, unofficial].
- Parameters: `max_depth`, `henyey_greenstein_g`, `le_scale`, `temperature_scale`, `temperature_offset`, `infinite_light` [S1, unofficial].
- Random number generator (RNG) [S1, unofficial].

**Outputs:**
- Accumulated radiance $L$ along the ray [S1, unofficial].

**Data Structures:**
- `MajorantTransmittanceSampler`: Manages sampling of interaction points using a majorant extinction coefficient $\sigma_M \ge \sigma_t(\mathbf{x})$ [S1, unofficial]. Stores current position, remaining distance, and step history [G].
- `Event`: Enum or integer representing {Null, Absorption, Scatter} [S1, unofficial].

**Steps:**

1. **Initialize:** Set accumulated radiance $L = 0$ and `terminated` flag to `false` [S1, unofficial]. Initialize `path_depth` = 0 [G].
2. **Ray-Volume Intersection:** Intersect the current ray with the volume grid [S1, unofficial]. If no intersection, break the depth loop [S1, unofficial].
3. **Sampler Initialization:** Initialize `MajorantTransmittanceSampler` with the intersection, RNG, density grid, accessor, and total extinction coefficient [S1, unofficial].
4. **Sample Points Loop:** Loop through sampled points from the majorant sampler [S1, unofficial].
    a. **Emission:** If temperature grid is available, sample temperature at the point, convert to Kelvin ($T_K = t_{norm} \times \text{temperature\_scale} + \text{temperature\_offset}$), and add emission contribution: $L \leftarrow L + p_a \times \text{le\_scale} \times \mathbf{L}_e$ [S1, unofficial].
    b. **Calculate Probabilities:** Compute absorption probability $p_a$ and scattering probability $p_s$ based on the medium properties at $\mathbf{x}$ and the ratio of coefficients to the majorant [G]. Compute null probability: $p_n = \max(1.0 - p_a - p_s, 0.0)$ [S1, unofficial].
    c. **Sample Event:** Sample a discrete event from {Null, Absorption, Scatter} with probabilities {p_n, p_a, p_s} using a uniform random number $u \in [0, 1)$ [S1, unofficial].
    d. **Null:** If the event is Null, log it and continue to the next iteration of the path loop [S1, unofficial].
    e. **Scatter:** If the event is Scatter, check if `max_depth` is reached [S1, unofficial]. If so, log termination and break [S1, unofficial]. Otherwise, recursively sample the scattered radiance $L \leftarrow L + \text{sample\_Ld}(...)$, sample a new direction using the Henyey-Greenstein phase function with parameter $g$, update the ray, log the scatter, and break the current loop iteration to continue the path [S1, unofficial].
    f. **Absorption:** If the event is Absorption, log it, mark as terminated, and break [S1, unofficial].
5. **Termination Check:** If the path terminates without scattering, break the outer loop [S1, unofficial].
6. **Infinite Light:** If the ray is not terminated (escapes to infinity), add infinite light contribution: $L \leftarrow L + \text{params.infinite\_light.xyz} \times \text{params.infinite\_light.multiplier}$ [S1, unofficial].
7. **Accumulate:** Add the accumulated radiance to the film buffer, scaled by `imaging_ratio`, and increment the sample count [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_tracing_discrete_event(ray_origin, ray_direction, params, grid, rng):
    L = [0.0, 0.0, 0.0]
    terminated = False
    path_depth = 0
    current_o = ray_origin
    current_d = ray_direction

    while not terminated and path_depth < params.max_depth:
        hit = intersect_volume_grid(current_o, current_d, grid.bbox)
        if not hit:
            break

        sampler = MajorantTransmittanceSampler(hit.t0, hit.t1, current_d, grid.density_grid, grid.extinction_grid, rng)

        while sampler.has_next():
            point, dist = sampler.next_sample()

            # Calculate extinction coefficients
            sigma_t = grid.get_extinction(point)
            sigma_a = grid.get_absorption(point)
            sigma_s = sigma_t - sigma_a
            sigma_M = sampler.majorant

            p_a = sigma_a / sigma_M
            p_s = sigma_s / sigma_M
            p_n = max(1.0 - p_a - p_s, 0.0)

            # Emission
            if grid.has_temperature():
                temp_norm = grid.get_temperature(point)
                temp_K = temp_norm * params.temperature_scale + params.temperature_offset
                le = blackbody_radiation_xyz(temp_K)
                L[0] += p_a * params.le_scale * le[0]
                L[1] += p_a * params.le_scale * le[1]
                L[2] += p_a * params.le_scale * le[2]

            # Sample event
            u = rng.uniform()
            if u < p_n:
                continue
            elif u < p_n + p_a:
                terminated = True
                break
            else:
                if path_depth >= params.max_depth:
                    terminated = True
                    break
                path_depth += 1
                new_direction = sample_henyey_greenstein(current_d, params.henyey_greenstein_g, rng)
                current_o = point
                current_d = new_direction
                # L accumulates across outer loop iterations [G]
                break

        if terminated:
            break

    if not terminated:
        L[0] += params.infinite_light[0] * params.infinite_light[1]
        L[1] += params.infinite_light[0] * params.infinite_light[1]
        L[2] += params.infinite_light[0] * params.infinite_light[1]

    return [x * params.imaging_ratio for x in L]

# Helper definitions
def intersect_volume_grid(origin, direction, bbox):
    # Returns hit object with t0, t1 or None if no intersection
    pass

class MajorantTransmittanceSampler:
    def __init__(self, t0, t1, direction, density_grid, extinction_grid, rng):
        self.t0 = t0
        self.t1 = t1
        self.direction = direction
        self.density_grid = density_grid
        self.extinction_grid = extinction_grid
        self.rng = rng
        self.majorant = max(extinction_grid.max(), 1e-6)
        self.current_t = t0
        self.has_next_called = False

    def has_next(self):
        return self.current_t < self.t1

    def next_sample(self):
        # Sample distance using majorant exponential distribution
        u = self.rng.uniform()
        delta_t = -math.log(1.0 - u) / self.majorant
        self.current_t += delta_t
        point = self.current_t * self.direction
        return point, delta_t

def blackbody_radiation_xyz(temperature_k):
    h = 6.62606957e-34
    c = 2.99792458e8
    kb = 1.3806488e-23
    # Simplified Planck's law for XYZ approximation
    lambda_m = 5.0e-7
    num = 2 * h * c * c
    den = (lambda_m ** 5) * (math.exp((h * c) / (lambda_m * kb * temperature_k)) - 1)
    return [num / den, num / den, num / den]

def sample_henyey_greenstein(direction, g, rng):
    # Standard Henyey-Greenstein phase function sampling
    pass
```

**Parameters:**
- `henyey_greenstein_g`: Anisotropy parameter for the Henyey-Greenstein phase function [S1, unofficial].
- `max_depth`: Maximum recursion depth for path tracing [S1, unofficial].
- `le_scale`: Scaling factor for emission [S1, unofficial].
- `temperature_scale`, `temperature_offset`: Parameters to convert normalized temperature samples to physical temperature (Kelvin) [S1, unofficial].
- `infinite_light`: Directional light source properties (direction, intensity, multiplier) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Ensure $p_n$ is clamped to $[0, 1]$ to avoid negative probabilities due to floating-point errors [G].
- **Sparse Media:** Null events are frequent in sparse media; skipping them efficiently is key [S1, unofficial].
- **Majorant:** The majorant must be $\ge \sigma_t(\mathbf{x})$ everywhere; overestimating increases null events and reduces efficiency [G].
- **Temperature Conversion:** Ensure temperature conversion is physically accurate if using blackbody radiation [G].

## Subtopics

### [Importance Sampling for Discrete Media](7052-importance-sampling-for-discrete-media.md)
Accelerates path tracing in discrete participating media (e.g., sand) by replacing small-step integration with large-step sampling guided by precomputed probability densities. It differs from standard discrete event sampling by using spatial precomputation to determine scattering locations at larger distances, thereby reducing the number of sampling steps and improving convergence speed. Unlike methods for continuous media, it relies on the statistical independence of particle positions characteristic of discrete media.

## Operating characteristics
- **Time Complexity:** Depends on the number of sampled points per ray segment and the depth of the path; null events allow skipping interactions, reducing the number of expensive scattering calculations in sparse media [S1, unofficial].
- **Memory Complexity:** Requires storage for the volume grid (density, temperature, extinction) and the majorant; the sampler state is minimal [G].
- **Performance:** More efficient than continuous Russian Roulette in sparse media due to explicit null event handling; performance degrades in dense media where null events are rare [S1, unofficial].
- **Strengths:** Explicit handling of absorption and null events; supports temperature-dependent emission; efficient in sparse media [S1, unofficial].
- **Limitations:** Requires pre-computation or access to majorant; less efficient in dense media compared to methods that do not sample null events [S1, unofficial].
- **When to Use:** Sparse participating media, temperature-dependent emission, when explicit control over absorption/scattering/null events is desired [S1, unofficial].
- **When Not to Use:** Dense media where null events are rare, or when computational resources for pre-computing majorants/probabilities are limited [G].

## Background
Originating from implementations like the unofficial volume path tracer by Shibodd [S1, unofficial], this method adapts path tracing for participating media by discretizing the interaction probabilities [S1, unofficial]. It builds on the principles of [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md) but introduces discrete event sampling for better control and efficiency in specific media types [S1, unofficial].

## Sources
[S1] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
