# Chromatic Null-Scattering Transmittance Estimation for Next Event Estimation

Also called: NEE Transmittance with Null Scattering, Shadow Ray Transmittance in Chromatic Media

## Overview
This method estimates the transmittance from a surface point to a light source in participating media with wavelength-dependent scattering properties using null-scattering (delta-tracking) [S1]. It differs from standard majorant sampling by selecting a specific RGB channel for the majorant at each step, allowing for tighter bounds per channel and reducing the variance caused by the global majorant in chromatic media [S1]. It is designed for Next Event Estimation (NEE) within volumetric path tracing [S1].

## How it works
**Inputs:**
- Current path point $p$.
- Light sample point $p'$.
- Scene geometry and medium properties (scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, total coefficient $\sigma_t = \sigma_s + \sigma_a$, null-scattering coefficient $\sigma_n$, majorant $\sigma_m$).
- Random number generator `rng`.
- `max_null_collisions`: Maximum iterations for the null-scattering loop (default 1000) [S1].

**Outputs:**
- Transmittance spectrum $T_{light}$.
- PDF for the null-scattering path $p_{trans\_nee}$.
- PDF for the direct direction PDF $p_{trans\_dir}$.

**Data Structures:**
- `accum_t`: Accumulated distance along the shadow ray (scalar).
- `channel`: Integer index 0, 1, or 2 corresponding to R, G, B.
- `majorant`: Vector of size 3, containing $\sigma_m$ for each channel.
- `sigma_n`: Vector of size 3, containing $\sigma_n$ for each channel.
- `sigma_t`: Vector of size 3, containing $\sigma_t$ for each channel.

**Algorithm:**

1.  Initialize $T_{light} = [1, 1, 1]$, $p_{trans\_nee} = [1, 1, 1]$, $p_{trans\_dir} = [1, 1, 1]$.
2.  Set `accum_t` = 0.
3.  Create a shadow ray from $p$ to $p'$.
4.  Intersect the scene to find the nearest intersection `isect`.
5.  Compute `next_t` = distance($p, p'$). If `isect` exists, `next_t` = distance($p, \text{isect.position}$).
6.  If the medium exists along the path:
    a. Sample $u \sim U[0,1]$ from `rng`.
    b. Determine `channel` = $\lfloor u \times 3 \rfloor$, clamped to $[0, 2]$.
    c. Initialize `iteration` = 0.
    d. Loop while True:
        i. If `majorant[channel]` $\le 0$, break.
        ii. If `iteration` $\ge$ `max_null_collisions`, break. [G] (Guard against numerical instability).
        iii. Sample $t = -\ln(1 - \text{next(rng)}) / \text{majorant[channel]}$.
        iv. Compute $dt = \text{next\_t} - \text{accum\_t}$.
        v. Update $\text{accum\_t} = \min(\text{accum\_t} + t, \text{next\_t})$.
        vi. If $t < dt$ (null-scattering event):
            1. Update $T_{light} \ *= \exp(-\text{majorant} \times t) \times \frac{\sigma_n}{\max(\text{majorant})}$.
            2. Update $p_{trans\_nee} \ *= \exp(-\text{majorant} \times t) \times \frac{\text{majorant}}{\max(\text{majorant})}$.
            3. Calculate $\text{real\_prob} = \frac{\sigma_t}{\text{majorant}}$.
            4. Update $p_{trans\_dir} \ *= \exp(-\text{majorant} \times t) \times \text{majorant} \times (1 - \text{real\_prob}) / \max(\text{majorant})$.
            5. If $\max(T_{light}) \le 0$, break. [G] (Optimization for $\sigma_n=0$).
        vii. Else (hit surface):
            1. Update $T_{light} \ *= \exp(-\text{majorant} \times dt)$.
            2. Update $p_{trans\_nee} \ *= \exp(-\text{majorant} \times dt)$.
            3. Update $p_{trans\_dir} \ *= \exp(-\text{majorant} \times dt)$.
            4. Break loop.
        viii. Increment `iteration`.
7.  Return $T_{light}$, $p_{trans\_nee}$, $p_{trans\_dir}$.

**Usage in Path Throughput:**
When updating the path throughput with the estimated transmittance, divide by the average PDF over the RGB channels:
$$ \text{current\_path\_throughput} \ *= \frac{T_{light}}{\text{avg}(p_{trans\_dir})} $$
This normalization is required because the PDFs are scaled by $\max(\text{majorant})$ for numerical robustness [S1].

**Parameters:**
- `max_null_collisions`: Default 1000 [S1].

**Edge Cases and Pitfalls:**
- **Numerical Instability:** Without the `max_null_collisions` limit, numerical errors can lead to unbounded loops [S1].
- **Numerical Robustness:** The division by $\max(\text{majorant})$ prevents values from blowing up when $\sigma_n$ is large relative to the majorant [S1].
- **Zero Transmittance:** If $\sigma_n = 0$, $T_{light}$ becomes 0; the loop breaks early to save computation [S1].

## Operating characteristics
- **Time Complexity:** Proportional to the number of null-scattering events, bounded by `max_null_collisions`.
- **Memory Complexity:** Constant, per-ray.
- **Strengths:** Handles chromatic media efficiently by using channel-specific majorants, reducing the overestimation of the global majorant.
- **Limitations:** Requires precomputed channel-specific majorants and null-scattering coefficients. The hard iteration limit is an approximation.

## Background
This method is described in the context of Volumetric Path Tracing homework assignments [S1]. It adapts null-scattering techniques for next-event estimation in participating media.

## Sources
[S1] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
