# Russian Roulette

Also called: Monte Carlo termination, Probabilistic Path Termination, RR termination, Russian Roulette Termination, Russian Roulette for Ray Marching Termination, Russian Roulette for Ray Termination, Russian Roulette for Volumetric Paths

## Overview
Russian Roulette is a randomized early-out technique used in Monte Carlo volume rendering to remove the statistical bias introduced by deterministic early termination (such as breaking a ray-marching loop when transmittance drops below a fixed threshold) [S1, unofficial][S2, unofficial][S3, unofficial]. It probabilistically terminates rays with low contribution weights while scaling up the weights of surviving paths to maintain an unbiased expectation of the final pixel color [S4, unofficial][S5, unofficial]. Unlike deterministic cutoffs which simply discard remaining contributions, Russian Roulette ensures that the expected value of the rendered image remains correct, albeit at the cost of increased variance (noise) [S3, unofficial][S5, unofficial].

## How it works
Russian Roulette is applied when the accumulated transmittance (or path throughput) becomes sufficiently small, indicating that the remaining contribution to the pixel is negligible but non-zero [S5, unofficial][S6]. The method replaces a hard cutoff with a probabilistic decision [S2, unofficial][S5, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Current accumulated transmittance/throughput $T$ (or weight $W$) [S4, unofficial][S5, unofficial][S6].
    *   Termination probability parameter $p_{term}$ (or continuation probability $p_{cont}$) [S4, unofficial][S6].
    *   Random number generator state [S1, unofficial][S5, unofficial].
*   **Outputs:**
    *   Boolean: `continue` or `terminate` [S5, unofficial].
    *   Updated transmittance/weight $T'$ or $W'$ [S4, unofficial][S5, unofficial].

### Data Structures
No specific complex data structures are required beyond standard scalar variables for transmittance/weight and a random number generator [G].

### Parameters
*   **Threshold ($T_{thresh}$):** The minimum transmittance value below which Russian Roulette is considered [S5, unofficial]. Common values are $10^{-3}$ [S5, unofficial].
*   **Termination Probability ($p_{term}$):** The probability of terminating the path [S4, unofficial]. In path tracing, this is often derived from the maximum color channel of the throughput [S4, unofficial][S6]. In ray marching, it is often a constant (e.g., 0.5) or derived from a compensation factor $D$ [S5, unofficial].
*   **Compensation Factor ($D$):** A value $> 1$ used in some formulations where $p_{term} = 1 - 1/D$ and $p_{cont} = 1/D$ [S1, unofficial][S5, unofficial].

### Algorithm Steps
1.  **Check Threshold:** If the current accumulated transmittance $T$ (or weight $W$) is greater than or equal to the threshold $T_{thresh}$, continue marching/tracing normally [S5, unofficial].
2.  **Determine Probability:** Set the continuation probability $p_{cont}$ (where $p_{term} = 1 - p_{cont}$) based on the formulation [S4, unofficial][S6].
    *   *Formulation A (Constant/Heuristic):* Set $p_{cont}$ to a constant (e.g., 0.5) [S5, unofficial].
    *   *Formulation B (Compensation Factor):* Set $p_{cont} = 1/D$ where $D > 1$ [S1, unofficial][S5, unofficial].
    *   *Formulation C (Path Tracing):* Set $p_{cont}$ to the maximum component value of the current throughput vector $\beta$ (or a scaled version thereof) [S6].
3.  **Generate Random Number:** Sample a uniform random number $\xi$ from $[0, 1]$ [S1, unofficial][S5, unofficial].
4.  **Probabilistic Decision:**
    *   If $\xi < p_{cont}$ (or $\xi > p_{term}$), continue the path [S4, unofficial][S5, unofficial][S6].
    *   Otherwise, terminate the path [S4, unofficial][S5, unofficial][S6].
5.  **Weight Compensation:**
    *   If continuing, scale the transmittance/weight to compensate for the probability of termination: $T' = T / p_{cont}$ (or $T' = T \cdot D$) [S1, unofficial][S4, unofficial][S5, unofficial][S6].
    *   If terminating, set $T' = 0$ (or $W' = 0$) [S4, unofficial][S5, unofficial].

### Pseudocode
The following pseudocode implements the standard Russian Roulette for volume ray marching, combining the threshold check and probabilistic termination [S5, unofficial][S6].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def russian_roulette(transmittance, threshold=1e-3, p_cont=0.5):
    """
    Applies Russian Roulette to the current transmittance.
    
    Args:
        transmittance: Current accumulated transmittance (Spectrum or float).
        threshold: Minimum transmittance to trigger RR.
        p_cont: Probability of continuing the path (0 < p_cont <= 1).
    
    Returns:
        (continue_marching: bool, new_transmittance: float)
    """
    # 1. Check if we are above the threshold
    if transmittance >= threshold:
        return True, transmittance

    # 2. Generate random number [G]
    r = random_float()

    # 3. Probabilistic decision
    if r < p_cont:
        # Survive: scale transmittance to compensate for termination probability
        # Avoid division by zero [G]
        if p_cont > 1e-6:
            new_transmittance = transmittance / p_cont
        else:
            new_transmittance = 0.0
        return True, new_transmittance
    else:
        # Die: terminate path
        return False, 0.0

def random_float():
    """Returns a uniform random number in [0, 1] [G]."""
    import random
    return random.random()
```

### Edge Cases and Pitfalls
*   **Bias from Simple Cutoff:** Simply breaking the loop when `transmittance < threshold` without compensation introduces significant bias by discarding the remaining volume contribution [S5, unofficial].
*   **Variance Increase:** While unbiased, Russian Roulette increases the variance (noise) of the image because it introduces binary outcomes (0 or scaled value) for paths that would have contributed small, deterministic amounts [S3, unofficial][S5, unofficial].
*   **Division by Zero:** Ensure the continuation probability is not zero to avoid division by zero when scaling the weight [G].

## Subtopics

### [Russian Roulette Early Ray Termination](13726-russian-roulette-early-ray-termination.md)
A sophisticated variant that probabilistically terminates rays based on accumulated opacity, replacing deterministic threshold-based cutoffs. It differs from standard early termination by allowing rays to continue with scaled weights even when opacity is high, thereby reducing computation while maintaining unbiased expectations through variance masking.

## Operating characteristics
*   **Time Complexity:** Reduces the average number of samples per ray by terminating low-contribution paths early [S4, unofficial].
*   **Memory Complexity:** No additional memory overhead beyond standard scalar variables [G].
*   **Performance:** Significant speedup in scenes with high variance or deep paths [S4, unofficial]. For example, rendering with Russian Roulette took 161s (unbiased), compared to 2556s without Russian Roulette (biased, max depth 200) in a cloud example [S4, unofficial].
*   **Strengths:** Unbiased results; effective for reducing computation in transparent or deep volumes [S2, unofficial][S4, unofficial].
*   **Limitations:** Increases noise (variance) [S3, unofficial].
*   **When to Use:** When using Monte Carlo integration for volume rendering, especially with path tracing or when deterministic early termination causes visible bias [S2, unofficial][S5, unofficial].
*   **When Not to Use:** When deterministic bias is acceptable for speed, or when the volume is very dense and transmittance drops to zero quickly anyway [S5, unofficial].

## Background
Russian Roulette originated in Monte Carlo particle-tracing methods to avoid simulating particles with low weights without biasing the results [S2, unofficial]. It was adapted for volume rendering to correct the bias introduced by early ray termination (cutoff) [S2, unofficial][S3, unofficial]. The technique is named after the dangerous game of chance, reflecting the probabilistic nature of path survival [S1, unofficial]. It is a standard component of unbiased path tracers and volume integrators [S6].

## Sources
[S1] "Volume Rendering for Developers" Group Read Part 1 [2023-03-30], https://www.youtube.com/watch?v=mZzMVD4LQvk, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S4] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S5] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S6] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
