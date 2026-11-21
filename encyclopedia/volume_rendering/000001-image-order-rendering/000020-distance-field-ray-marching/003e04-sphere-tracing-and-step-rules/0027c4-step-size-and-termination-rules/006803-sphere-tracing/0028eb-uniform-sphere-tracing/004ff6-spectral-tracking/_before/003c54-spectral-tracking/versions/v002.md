# Spectral Tracking

Also called: Single-Distribution Spectral Tracking, Vectorized Weighted Delta Tracking

## Overview
Spectral tracking is a volume rendering technique that efficiently handles per-wavelength optical properties by decomposing the medium's scattering and absorption coefficients into spectral components, using a single majorant density to drive the sampling process for all wavelengths simultaneously [S1, unofficial][S2]. Unlike naive delta tracking which handles each wavelength separately, spectral tracking invokes the tracker only once for all wavelengths, computing a spectrally resolved collision weight at each collision to counteract the discrepancy between the single sampling distribution and the true per-wavelength free-path distribution [S2]. This reduces the number of tracker invocations and leverages vectorized operations, making it particularly effective for spectrally heterogeneous media [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A Volume Scene Definition (`vsdf`) containing object properties, density volume, emission volume, and a scalar scatter ratio; a maximum ray distance (`max_distance`); a random number generator state (`rng`); and a ray defined by origin `o` and direction `d`.
*   **Output:** A pair containing the path length traveled and a spectral contribution vector (typically RGB) representing the accumulated transmittance and scattering contribution.

**Data Structures:**
*   `vsdf`: Object containing `object`, `density_vol`, `emission_vol`, and `scatter` (scalar ratio of scattering to total extinction).
*   `wD`: A vector of size $N_\lambda$ (number of wavelengths) initialized to ones, representing the spectral weight. In standard implementations, $N_\lambda = 3$ for RGB primaries [S2].
*   `sigma_t`, `sigma_s`, `sigma_a`, `sigma_n`: Vectors of size $N_\lambda$ representing total extinction, scattering, absorption, and non-scattering (null) coefficients at the current position.
*   `Pt`, `Ps`, `Pa`, `Pn`: Normalized probabilities for total, scattering, absorption, and null events, respectively.

**Parameters:**
*   $N_\lambda$: Number of wavelengths. Typically 3 (RGB) [S2].
*   $\bar{\mu}(x)$: The free-path-sampling coefficient (majorant). It must bound the maximum value of $\mu_t(x, \lambda)$ across all wavelengths $\lambda$ to keep estimation variance low [S2].
*   `max_distance`: Maximum distance the ray travels within the medium.

**Algorithm Steps:**

1.  **Initialization:**
    *   Retrieve `object`, `density_vol`, `emission_vol`, and `scatter` from `vsdf`.
    *   Calculate `max_density` from `density_vol->max_voxel * object->density_mult`.
    *   Set `majorant_density` = `max_density`.
    *   Set `imax_density` = $1.0 / \text{max\_density}$.
    *   Initialize `path_length` = 0.0.
    *   Initialize `tr` (transmittance) = $\vec{1}$ (vector of ones).
    *   Initialize `f` (throughput factor) = $\vec{1}$.
    *   Calculate `sigma_s` = `vsdf.scatter` * `max_density`.
    *   Calculate `Ps` = `sigma_s` * `imax_density`.
    *   Initialize spectral weight vector `wD` = $(1, \dots, 1)^{N_\lambda}$ [S2].

2.  **Main Loop (While True):**
    *   **Sample Distance:** Generate a uniform random number $\zeta \in [0, 1)$. Sample tentative distance $t = -\ln(1-\zeta) / \bar{\mu}$. In the implementation, this is often combined with the path length update: `path_length` -= $\log(1 - \text{rand1f}(rng)) \times \text{imax\_density}$ [S1, unofficial].
    *   **Termination Check:** If `path_length` >= `max_distance`, break the loop and return `{max_distance, f}` [S1, unofficial].
    *   **Update Position:** Calculate `current_pos` = `ray.o` + `path_length` * `ray.d` [S1, unofficial].
    *   **Evaluate Coefficients:**
        *   Evaluate total extinction coefficient vector $\sigma_t$ at `current_pos` using `eval_vpt_density(vsdf, current_pos)` [S1, unofficial].
        *   Calculate non-scattering extinction vector $\sigma_n = \text{majorant\_density} - \sigma_t$ [S1, unofficial].
        *   Calculate absorption coefficient vector $\sigma_a = \sigma_t - \sigma_s$ [S1, unofficial].
    *   **Calculate Probabilities:**
        *   $P_t = \sigma_t \times \text{imax\_density}$ [S1, unofficial].
        *   $P_n = \sigma_n \times \text{imax\_density}$ [S1, unofficial].
        *   $P_a = 1.0 - P_s - P_n$ [S1, unofficial]. Note: $P_s$ is constant from initialization.
    *   **Update Transmittance:** `tr` *= $\exp(-\sigma_t \times \text{path\_length})$ [S1, unofficial].
    *   **Event Selection:**
        *   Generate a random scalar $\xi = \text{rand1f}(rng)$.
        *   **Absorption Check:** If $\max(P_a) > \xi$:
            *   Calculate contribution vector `contrib` = $\sigma_a / (\text{majorant\_density} \times P_a)$ [S1, unofficial].
            *   Normalize: `contrib` /= $\max(\text{contrib})$ [S1, unofficial].
            *   Set `vsdf.event` = `EVENT_ABSORB`.
            *   Set `vsdf.density` = $\sigma_t$.
            *   Return `{path_length, tr * contrib}` [S1, unofficial].
        *   **Scattering Check:** Else if $\max(P_t) > \xi$:
            *   Calculate contribution vector `contrib` = $\sigma_s / (\text{majorant\_density} \times P_s)$ [S1, unofficial].
            *   Normalize: `contrib` /= $\max(\text{contrib})$ [S1, unofficial].
            *   Set `vsdf.event` = `EVENT_SCATTER`.
            *   Set `vsdf.density` = $\sigma_t$.
            *   Return `{path_length, tr * contrib}` [S1, unofficial].
        *   **Null Collision (Non-Interaction):**
            *   Calculate `contrib` = $\sigma_n$ [S1, unofficial].
            *   Normalize: `contrib` /= $\max(\text{contrib})$ [S1, unofficial].
            *   Update throughput factor: `f` *= `contrib` [S1, unofficial].
            *   *Note on Weight Vector:* The source algorithm description [S2] indicates that for null collisions, the spectral weight vector should be updated: $w_D \leftarrow w_D \times \frac{\mu_n(x)}{\bar{\mu} P_n(x)}$. However, the provided implementation code [S1, unofficial] does not explicitly update `wD` in the null branch, instead updating `f`. This discrepancy suggests `f` may serve as a proxy for the accumulated weight in this specific implementation, or the implementation is simplified. For strict adherence to the "Vectorized Weighted Delta Tracking" definition [S2], the weight vector update should be applied. Given the instruction to implement from the text, and the explicit code in [S1, unofficial], the code's behavior is documented, but the theoretical requirement from [S2] is noted. *Correction:* The code in [S1, unofficial] returns immediately on absorption or scattering. The null case updates `f` and continues. The spectral weight `wD` is initialized but not used in the provided C++ snippet [S1, unofficial]. This implies the provided code might be a simplified or specific variant where spectral weights are handled differently or the snippet is incomplete regarding the full spectral weight propagation described in [S2]. However, the prompt asks to implement from the article. The article [S1, unofficial] provides the code. The article [S2] provides the theory. The code in [S1, unofficial] is labeled "eval_spectral_tracking". I will follow the code in [S1, unofficial] for the step-by-step logic as it is the concrete implementation, but note the theoretical weight update from [S2] if necessary for correctness. Actually, looking closely at [S1, unofficial], it returns `tr * contrib`. `tr` is a vector. `contrib` is a vector. The spectral nature is preserved in the vectors. The `wD` vector in [S2] is for *weighted* tracking where the weight is accumulated. The code in [S1, unofficial] seems to use a different formulation where the contribution is calculated directly at the event. I will stick to the code in [S1, unofficial] as the primary implementation source for the steps, as it is self-contained.

    *   **Loop Continuation:** If null collision, continue to next iteration.

3.  **Final Return:** If the loop completes (ray exits max distance), return `{max_distance, f}` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Variance Growth:** The variance of the spectral tracker depends on the variance of the spectrally resolved collision weights. If probabilities are poorly chosen, weights can deviate significantly from 1, leading to geometric growth of path throughput [S2].
*   **Majorant Tightness:** The majorant density $\bar{\mu}$ should be as close as possible to the maximum $\mu_t(x, \lambda)$ to minimize variance [S2].
*   **Implementation Discrepancy:** The provided code [S1, unofficial] does not explicitly maintain or update the spectral weight vector `wD` as described in the theoretical algorithm [S2]. It uses a scalar random check against vector probabilities. This may be a simplification or a specific implementation choice. Implementers should be aware that strict adherence to the weighted delta tracking theory [S2] requires maintaining `wD`.

## Subtopics

### Spectral and Decomposition Tracking Combined
This variant combines spectral tracking with decomposition tracking to further optimize performance for subsurface scattering in heterogeneous volumes without acceleration structures [S2]. It computes control and free-path-sampling coefficients based on approximate minimum and maximum values over the entire medium via random sampling in a pre-pass [S2]. In the translucent dragon scene, this combination reduced the Light Transport Unbiased Variance (LTUV) by 2.70× over delta tracking (spectral alone) and further by 2.05× (combined), yielding a total 5.54× lower LTUV than delta tracking and 1.65× lower RMSE [S2]. It also reduced total render time by 22% compared to delta tracking [S2]. The decrease in render time with spectral tracking is attributed to different path distributions and lower throughputs allowing Russian roulette to terminate paths earlier [S2].

## Operating characteristics
*   **Time Complexity:** Reduces the number of tracker invocations compared to handling each wavelength separately [S2]. In the translucent dragon scene, spectral tracking reduced render time from 133 core seconds (delta tracking) to 112 core seconds [S2]. Combined with decomposition tracking, it further reduced time to 88 core seconds [S2].
*   **Memory Complexity:** Requires storage for spectral vectors (size $N_\lambda$) for coefficients and weights. For RGB, this is minimal overhead.
*   **Performance:** In the translucent dragon scene, spectral tracking reduced LTUV by 2.70× over delta tracking [S2]. Combined techniques yielded 1.65× lower RMSE than delta tracking [S2].
*   **Strengths:** Efficiently handles spectrally resolved collision coefficients; reduces tracker invocations; suitable for subsurface scattering in heterogeneous media without acceleration structures [S2].
*   **Limitations:** Variance can grow if the majorant is loose or probabilities are poorly chosen; efficiency of decomposition tracking (when combined) depends on tight bounds of extinction coefficients, which is challenging in highly heterogeneous media [S2].
*   **When to Use:** Spectrally heterogeneous media, subsurface scattering, when acceleration structures are not available or impractical [S2].
*   **When Not to Use:** Highly heterogeneous media where tight bounds for decomposition are hard to find; if strict variance control is required and the majorant cannot be tightly bounded [S2].

## Background
Spectral tracking was introduced as part of the "Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes" paper from Disney Animation [S2]. It was developed to efficiently handle spectrally resolved collision coefficients by invoking the tracker only once for all wavelengths, using a single distribution and reweighting [S2]. The technique uses a spectral basis, typically RGB primaries, but holds for arbitrary counts [S2]. It builds on the principles of delta tracking [S2].

## Sources
[S1] Regarding spectral tracking code wrong (eval_spectral_tracking), https://github.com/RickyMexx/yocto-gl-vpt/issues/4, fetched 2026-10-28 (unofficial)
[S2] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
