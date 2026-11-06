# Russian Roulette
Also called: Break From the Ray Marching Loop When Opaque (Optimization), Monte Carlo termination, Probabilistic Path Termination, RR termination, Russian Roulette for Ray Marching Termination, Russian Roulette for Volumetric Paths, Volumetric path termination

## Overview
Russian Roulette is a probabilistic technique used to terminate volumetric paths or ray marching loops when their contribution to the final image becomes negligible, thereby reducing computational cost while maintaining statistical unbiasedness [S1, S2, unofficial][S3]. Unlike deterministic early termination which clamps the result and introduces bias, Russian Roulette scales the weight of surviving paths to compensate for the probability of termination, ensuring the expected value remains unchanged [S1, S2, unofficial]. It is applied during the ray marching or path integration loop after opacity or transmittance updates [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current path throughput/weight $W$ (or transmittance $T$), random number generator, current path depth [S1, S2, unofficial][S3].
- Output: Updated weight $W'$ (or $T'$) and a boolean indicating whether the path continues or terminates [S1, S2, unofficial].

**Data Structures:**
- Random number generator producing uniform samples $\xi \in [0, 1]$ [S1, S2, unofficial].
- Accumulated throughput/transmittance variable (scalar or spectral vector) [S1, unofficial][S3].
- Path depth counter [S3].

**Parameters:**
- Termination probability $p_{rr}$: The probability of killing the path, often derived from the weight or transmittance [S1, S2, unofficial].
- Threshold: A minimum value for transmittance/weight below which RR is considered (e.g., $10^{-3}$) [S2, unofficial].
- Minimum depth: Current path depth, used to prevent termination at the first bounce [S3].

**Steps:**
1. Evaluate the current contribution weight $W$ (or transmittance $T$) after a scattering event or step [S1, S2, unofficial].
2. Determine the termination probability $p_{rr}$ (or survival probability $1-p_{rr}$). Common strategies include: using the maximum component of the spectral weight/transmittance [S1, unofficial][S3]; using a fixed constant (e.g., $0.5$) if the weight is below a threshold [S2, unofficial]; or using the transmittance value itself for pure transmittance calculations [S1, unofficial].
3. Generate a uniform random number $\xi \in [0, 1]$ [S1, S2, unofficial].
4. Compare $\xi$ with $p_{rr}$: if $\xi \geq p_{rr}$ (or $\xi > 1/d$ in some formulations where $d=1/p_{rr}$), terminate the path and set $W' = 0$ [S1, S2, unofficial]. Otherwise, continue the path and scale the weight: $W' = W / p_{rr}$ [S1, S2, unofficial].

**Formulas:**
The expected value of the updated weight $W'$ is:
$$
E[W'] = (1 - p_{rr}) \cdot 0 + p_{rr} \cdot \frac{E[W]}{p_{rr}} = E[W]
$$
This confirms the estimator remains unbiased [S1, unofficial].

In spectral rendering, the termination probability $q$ is often derived from the maximum component of the scaled beta ($\beta$):
$$
q = \max(0, 1 - \beta_{\text{scaled}}.\text{MaxComponentValue}())
$$
where $\beta_{\text{scaled}}$ may include transmission scaling factors [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

class Spectral:
    def __init__(self, components):
        self.components = components
    def max_component(self):
        return max(self.components)
    def divide(self, scalar):
        return Spectral([c / scalar for c in self.components])

def russian_roulette(weight, min_threshold=1e-3, min_depth=1, current_depth=1):
    """
    Applies Russian Roulette to a volumetric path.
    
    Args:
        weight: Current throughput/transmittance (scalar or Spectral).
        min_threshold: Minimum weight to consider termination.
        min_depth: Minimum path depth before allowing termination.
        current_depth: Current path depth.
        
    Returns:
        new_weight: Scaled weight if continuing, 0 if terminated.
        continue_path: Boolean indicating if the path continues.
    """
    # Do not terminate at the first bounce or if weight is high
    if current_depth < min_depth:
        return weight, True

    # Determine termination probability based on weight
    if isinstance(weight, Spectral):
        max_comp = weight.max_component()
        if max_comp < min_threshold:
            # Probability of continuing is max_comp
            # Probability of terminating q = 1 - max_comp
            q = max(0.0, 1.0 - max_comp)
        else:
            return weight, True
    else:
        # Strategy 2: Fixed probability if below threshold [S2, unofficial]
        if weight < min_threshold:
            # d=2 implies continueProb=0.5, so p_rr (termination) = 0.5
            p_rr = 0.5
            q = p_rr
        else:
            return weight, True

    # Sample
    u_rr = random.random()

    # Terminate if u_rr < q (where q is prob of termination)
    if u_rr < q:
        return 0.0, False
    else:
        # Scale weight by 1 / (1 - q) which is 1 / continue_prob
        # If q = 1 - max_comp, then 1-q = max_comp
        new_weight = weight.divide(1.0 - q) if isinstance(weight, Spectral) else weight / (1.0 - q)
        return new_weight, True
```

**Edge Cases and Pitfalls:**
- **Bias from Simple Break:** Simply breaking the loop when transparency drops below a threshold without compensation removes the contribution of the volume beyond that point, introducing bias [S2, unofficial].
- **Zero Intersection:** If a ray does not intersect the volume or the intersection length is zero, transmission is set to 1.0 and color contribution is 0, avoiding unnecessary computation [S2, unofficial].
- **Spectral Balance:** In spectral rendering, using the maximum component for termination probability helps maintain spectral balance [S3].
- **Depth Constraint:** Termination should not occur at depth 1 to ensure the first scattering event is always accounted for [S3].

## Subtopics

### [Empty-space skipping](1002-empty-space-skipping.md)
Avoids sampling empty regions by subdividing volumes. Choose when volumes have large empty spaces.

### [Early ray termination](1019-early-ray-termination.md)
Stops marching when opacity exceeds a threshold. Choose for simple opacity culling without unbiased guarantees.

### [Bounding Volume Culling](1239-bounding-volume-culling.md)
Uses BVH to skip non-intersecting volumes. Choose for complex scenes with many volumes.

### [Early Stop](1790-early-stop.md)
General term for stopping integration early. Choose when performance is critical and bias is acceptable.

### [Conservative Early Ray Termination](1949-conservative-early-ray-termination.md)
A safer variant of early termination. Choose when visual quality must be preserved with minimal bias.

## Operating characteristics
- **Time Complexity:** Reduces the average number of ray marching steps, especially in dense volumes where transmittance drops quickly [S2, unofficial].
- **Memory Complexity:** No additional memory overhead beyond random number state [G].
- **Measured Performance:** In a cloud example, rendering with Russian Roulette took 161s (unbiased) compared to 2556s without it (biased, max depth 200) [S1, unofficial].
- **Scaling:** More effective in dense volumes where transparency drops rapidly [S2, unofficial].
- **Strengths:** Maintains unbiased results while significantly speeding up rendering in scenes with high variance or dense media [S1, unofficial].
- **Limitations:** Adds variance due to the random termination; requires careful tuning of termination probability [S1, unofficial].
- **When to Use:** In path tracing or ray marching for volumes where paths can become very long or contribute negligibly [S1, S2, unofficial].

## Background
Russian Roulette originates from Monte Carlo simulation techniques, where it is used to terminate particle paths that contribute little to the final result [S2, unofficial]. In computer graphics, it was adapted for volumetric rendering to handle the long paths often encountered in participating media [S1, unofficial]. The technique is widely used in modern path tracers like PBRT to ensure unbiased rendering of complex volumetric scenes [S2, unofficial][S3].

## Sources
[S1] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S2] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
