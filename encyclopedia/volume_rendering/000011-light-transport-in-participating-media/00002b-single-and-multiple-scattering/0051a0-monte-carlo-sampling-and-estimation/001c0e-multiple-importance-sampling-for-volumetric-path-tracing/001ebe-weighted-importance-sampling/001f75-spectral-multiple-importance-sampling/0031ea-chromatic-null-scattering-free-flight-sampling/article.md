# Chromatic Null-Scattering Free-Flight Sampling

Also called: One-sample multiple importance sampling for chromatic media, RGB channel sampling for null scattering

## Overview
This method extends null-scattering free-flight sampling to handle spectrally varying (chromatic) participating media by sampling a single color channel (R, G, or B) for distance determination while averaging the probability density functions (PDFs) across all channels to maintain unbiasedness [S1]. It solves the problem where a single sampling PDF based on one wavelength is a poor match or invalid for other wavelengths, causing high variance in spectral path tracing [S1]. It differs from standard monochromatic null-scattering by introducing channel selection logic and PDF averaging, and from full spectral sampling by reducing the per-step computational cost to a single channel's majorant evaluation [S1].

## How it works
**Inputs:**
- Ray origin, direction, and $t_{hit}$ (distance to next surface intersection).
- Medium properties: extinction coefficient $\sigma_t$ (vector of size 3, e.g., RGB), scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$.
- Majorant $\sigma_m$ (vector of size 3), precomputed as the element-wise maximum of $\sigma_t$ over the medium's domain [S1].
- Random number generator `rng`.
- `max_null_collisions`: integer limit for loop termination (default 1000) [S1].

**Outputs:**
- `scatter`: boolean, true if a real particle interaction occurred.
- `transmittance`: spectrum (vector of size 3), accumulated transmittance.
- `trans_dir_pdf`: spectrum (vector of size 3), PDF for free-flight sampling (used in path throughput).
- `trans_nee_pdf`: spectrum (vector of size 3), PDF for next event estimation (used in light transmittance).

**Data Structures:**
- Vectors are 3-element arrays representing R, G, B channels.
- `make_const_spectrum(v)`: creates a vector $[v, v, v]$.
- `max(v)`: returns the maximum element of vector $v$.
- `avg(v)`: returns the arithmetic mean of vector elements $\frac{1}{3}\sum v_i$.
- `clamp(x, min, max)`: clamps $x$ to $[min, max]$.

**Formulas:**
- Real particle probability: $p_{real} = \frac{\sigma_t}{\sigma_m}$ (element-wise division) [S1].
- Null-scattering coefficient: $\sigma_n = \sigma_m (1 - p_{real}) = \sigma_m - \sigma_t$ [S1].
- Distance sample: $t = -\frac{\log(1 - u)}{\sigma_m[channel]}$, where $u \in [0, 1)$ [S1].

**Parameters:**
- `max_null_collisions`: Default 1000 [S1].

**Algorithm Steps:**
1. Initialize `scatter = False`, `transmittance = [1, 1, 1]`, `trans_dir_pdf = [1, 1, 1]`, `trans_nee_pdf = [1, 1, 1]` [S1].
2. If the ray is inside the medium:
   a. Retrieve $\sigma_m$ for the current ray segment [S1].
   b. Sample $u \sim \text{Uniform}(0, 1)$ [S1].
   c. Determine channel index: $c = \text{clamp}(\lfloor u \times 3 \rfloor, 0, 2)$ [S1].
   d. Initialize `accum_t = 0`, `iteration = 0` [S1].
3. Loop indefinitely:
   a. If $\sigma_m[c] \le 0$, break (no scattering possible in selected channel) [S1].
   b. If `iteration` $\ge$ `max_null_collisions`, break (prevent unbounded loops due to numerical error) [S1].
   c. Sample distance $t = -\frac{\log(1 - \text{next}(rng))}{\sigma_m[c]}$ [S1].
   d. Compute remaining distance to surface: $dt = t_{hit} - \text{accum\_t}$ [S1].
   e. Update accumulated distance: $\text{accum\_t} = \min(\text{accum\_t} + t, t_{hit})$ [S1].
   f. If $t < dt$ (interaction occurs before surface):
      i. Compute $p_{real} = \frac{\sigma_t}{\sigma_m}$ [S1].
      ii. Sample $v \sim \text{Uniform}(0, 1)$ [S1].
      iii. If $v < p_{real}[c]$ (hit real particle):
           - Set `scatter = True` [S1].
           - Update `transmittance` $\leftarrow$ `transmittance` $\times \frac{\exp(-\sigma_m \cdot t)}{\max(\sigma_m)}$ [S1].
           - Update `trans_dir_pdf` $\leftarrow$ `trans_dir_pdf` $\times \frac{\exp(-\sigma_m \cdot t) \cdot \sigma_m[c] \cdot p_{real}[c]}{\max(\sigma_m)}$ [S1].
           - Break loop [S1].
      iv. Else (hit fake/null particle):
           - Update `transmittance` $\leftarrow$ `transmittance` $\times \frac{\exp(-\sigma_m \cdot t) \cdot \sigma_n[c]}{\max(\sigma_m)}$ [S1].
           - Update `trans_dir_pdf` $\leftarrow$ `trans_dir_pdf` $\times \frac{\exp(-\sigma_m \cdot t) \cdot \sigma_m[c] \cdot (1 - p_{real}[c])}{\max(\sigma_m)}$ [S1].
           - Update `trans_nee_pdf` $\leftarrow$ `trans_nee_pdf` $\times \frac{\exp(-\sigma_m \cdot t) \cdot \sigma_m[c]}{\max(\sigma_m)}$ [S1].
   g. Else (reach surface, $t \ge dt$):
      - Update `transmittance` $\leftarrow$ `transmittance` $\times \exp(-\sigma_m \cdot dt)$ [S1].
      - Update `trans_dir_pdf` $\leftarrow$ `trans_dir_pdf` $\times \exp(-\sigma_m \cdot dt)$ [S1].
      - Update `trans_nee_pdf` $\leftarrow$ `trans_nee_pdf` $\times \exp(-\sigma_m \cdot dt)$ [S1].
      - Break loop [S1].
   h. Increment `iteration` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chromatic_null_scattering_free_flight(ray, medium, rng, max_null_collisions=1000):
    scatter = False
    transmittance = [1.0, 1.0, 1.0]
    trans_dir_pdf = [1.0, 1.0, 1.0]
    trans_nee_pdf = [1.0, 1.0, 1.0]

    if in_medium(ray.origin, medium):
        sigma_m = get_majorant(medium, ray)
        u = next(rng)
        channel = clamp(int(u * 3), 0, 2)
        accum_t = 0
        iteration = 0

        while True:
            if sigma_m[channel] <= 0:
                break
            if iteration >= max_null_collisions:
                break

            t = -log(1 - next(rng)) / sigma_m[channel]
            dt = ray.t_hit - accum_t
            accum_t = min(accum_t + t, ray.t_hit)

            if t < dt:
                real_prob = sigma_m[0] / sigma_m[0] # Placeholder: sigma_t / sigma_m
                # Note: sigma_t must be retrieved from medium at current position
                sigma_t = get_sigma_t(medium, ray.origin + ray.direction * accum_t)
                real_prob = sigma_t / sigma_m

                if next(rng) < real_prob[channel]:
                    scatter = True
                    transmittance = [t * (exp(-sigma_m[i] * t) / max(sigma_m)) for i, t in enumerate(transmittance)]
                    trans_dir_pdf = [p * (exp(-sigma_m[i] * t) * sigma_m[i] * real_prob[i] / max(sigma_m)) for i, p in enumerate(trans_dir_pdf)]
                    break
                else:
                    sigma_n = [sigma_m[i] * (1 - real_prob[i]) for i in range(3)]
                    transmittance = [t * (exp(-sigma_m[i] * t) * sigma_n[i] / max(sigma_m)) for i, t in enumerate(transmittance)]
                    trans_dir_pdf = [p * (exp(-sigma_m[i] * t) * sigma_m[i] * (1 - real_prob[i]) / max(sigma_m)) for i, p in enumerate(trans_dir_pdf)]
                    trans_nee_pdf = [p * (exp(-sigma_m[i] * t) * sigma_m[i] / max(sigma_m)) for i, p in enumerate(trans_nee_pdf)]
            else:
                transmittance = [t * exp(-sigma_m[i] * dt) for i, t in enumerate(transmittance)]
                trans_dir_pdf = [p * exp(-sigma_m[i] * dt) for i, p in enumerate(trans_dir_pdf)]
                trans_nee_pdf = [p * exp(-sigma_m[i] * dt) for i, p in enumerate(trans_nee_pdf)]
                break

            iteration += 1

    return scatter, transmittance, trans_dir_pdf, trans_nee_pdf
```
*[G] The pseudocode above assumes helper functions `get_sigma_t` and `get_majorant` which are standard in volume rendering implementations. The element-wise operations are explicitly written out for clarity. The formula for `sigma_n` is derived from $\sigma_m(1 - \sigma_t/\sigma_m)$.]*

**Edge Cases and Pitfalls:**
- **Unbounded Loops:** Numerical errors can cause infinite loops if the majorant is very small or zero. The loop is terminated when `iteration` reaches `max_null_collisions` (default 1000) [S1].
- **Numerical Overflow:** Values can blow up to infinity if not normalized because $\sigma_n$ can be large. Both transmittance and PDFs are divided by $\max(\sigma_m)$ to improve numerical robustness [S1].
- **Zero Majorant:** If $\sigma_m[channel] \le 0$, the loop breaks immediately to avoid division by zero [S1].

## Operating characteristics
- **Time Complexity:** $O(k)$ per free-flight step, where $k$ is the number of null-scattering events before a real interaction or surface hit. $k$ depends on the ratio $\sigma_t / \sigma_m$.
- **Memory Complexity:** $O(1)$ additional memory per ray, storing vectors of size 3.
- **Performance:** Reduces variance compared to monochromatic sampling in chromatic media by ensuring the PDF is non-zero and well-matched for all wavelengths [S1].
- **Strengths:** Unbiased spectral rendering; handles arbitrary chromatic media; robust against numerical instability via normalization [S1].
- **Limitations:** Requires precomputed majorants; increased computational cost per step due to vector operations and channel selection; potential for unbounded loops if not capped [S1].
- **When to Use:** Spectral path tracing with chromatic participating media where variance from monochromatic sampling is unacceptable [S1].
- **When Not to Use:** Monochromatic media (use standard null-scattering); real-time rendering where performance is critical and chromatic accuracy is not required [S1].

## Background
- **Origin:** Introduced in the context of volumetric path tracing for chromatic media, specifically addressing the limitations of monochromatic null-scattering in spectral rendering [S1].
- **History:** Developed as part of the broader effort to enable unbiased spectral path tracing, building on null-scattering techniques [S1].
- **Usage:** Used in offline spectral renderers for accurate simulation of colored participating media [S1].

## Sources
[S1] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
