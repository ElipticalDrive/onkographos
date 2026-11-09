# Multiple Octaves Phase Function
## Overview
The Multiple Octaves Phase Function is a composite model that combines multiple Henyey-Greenstein lobes with varying asymmetry parameters and weights to create complex scattering patterns, specifically a two-lobed effect [S1, unofficial]. It addresses the limitation of single-parameter phase functions by allowing multi-lobed distributions without requiring more expensive physical models [S1, unofficial]. It sets itself apart from the sibling [Cornette and Shanks Phase Function](1710-cornette-and-shanks-phase-function.md) by using an iterative octave-based construction rather than a fixed analytical approximation [S1, unofficial].
## How it works
**Inputs:**
- Scattering angle cosine $\cos \theta = \omega_i \cdot \omega_o$ [G].
- Extinction coefficient $\sigma_t$ and density $\rho$ for attenuation context [S1, unofficial].

**Outputs:**
- Accumulated radiance contribution $L$ [S1, unofficial].

**Data Structures:**
- No persistent data structures. The method uses local scalar variables for the loop state [S1, unofficial].

**Algorithm:**
The method executes a fixed 4-octave loop [S1, unofficial]. Each octave mixes two Henyey-Greenstein functions with scaled asymmetry parameters [S1, unofficial].
1. Initialize accumulators: radiance $L = 0$, attenuation factor $a = 1.0$, contribution weight $b = 1.0$, and phase scale factor $c = 1.0$ [S1, unofficial].
2. Loop $i$ from 0 to 3 (4 iterations) [S1, unofficial]:
   a. Calculate asymmetry parameters: $g_1 = -0.1 \times c$ and $g_2 = 0.3 \times c$ [S1, unofficial].
   b. Evaluate two Henyey-Greenstein phase functions $p_{HG}(\cos \theta, g)$ [G].
   c. Mix the lobes with weights 0.3 and 0.7: $p_{octave} = 0.3 \times p_1 + 0.7 \times p_2$ [S1, unofficial].
   d. Calculate attenuation: $attenuation = \exp(-\sigma_t \times \rho \times a)$ [S1, unofficial].
   e. Add to radiance: $L += b \times p_{octave} \times attenuation$ [S1, unofficial].
   f. Update factors for next octave: $a \leftarrow a \times 0.2$, $b \leftarrow b \times 0.5$, $c \leftarrow c \times 0.5$ [S1, unofficial].
3. Return accumulated radiance $L$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def heney_greenstein(cos_theta, g):
    """Standard Henyey-Greenstein phase function [G]."""
    numerator = 1.0 - g * g
    denominator = 1.0 + g * g - 2.0 * g * cos_theta
    if denominator < 1e-8:
        denominator = 1e-8
    return numerator / (4.0 * math.pi * (denominator ** 1.5))

def multiple_octaves_phase_function(cos_theta, sigma_t, density):
    """Implements the 4-octave composite phase function from [S1, unofficial]."""
    radiance = 0.0
    a = 1.0
    b = 1.0
    c = 1.0

    for _ in range(4):
        g1 = -0.1 * c
        g2 = 0.3 * c

        p1 = heney_greenstein(cos_theta, g1)
        p2 = heney_greenstein(cos_theta, g2)

        phase_octave = 0.3 * p1 + 0.7 * p2

        attenuation = math.exp(-sigma_t * density * a)

        radiance += b * phase_octave * attenuation

        a *= 0.2
        b *= 0.5
        c *= 0.5

    return radiance
```

**Parameters:**
- Number of octaves: 4 (fixed) [S1, unofficial].
- Lobe weights: 0.3 (backward/negative $g$) and 0.7 (forward/positive $g$) [S1, unofficial].
- Asymmetry scaling: $g_1 = -0.1c$, $g_2 = 0.3c$ [S1, unofficial].
- Attenuation decay: $a$ multiplies by 0.2 per octave [S1, unofficial].
- Contribution decay: $b$ multiplies by 0.5 per octave [S1, unofficial].
- Phase scale decay: $c$ multiplies by 0.5 per octave [S1, unofficial].

**Edge Cases:**
- If $\sigma_t$ or density is 0, attenuation is 1.0 [S1, unofficial].
- If $g$ approaches $\pm 1$, the Henyey-Greenstein function becomes singular; implementation should clamp or handle delta spikes [G].
- The function returns an accumulated radiance contribution rather than a strictly normalized phase function $p(\omega_o, \omega_i)$ where $\int p d\omega = 1$ [S1, unofficial].

## Subtopics

### [Dual-Lobe Henyey-Greenstein Phase Function](5231-dual-lobe-henyey-greenstein-phase-function.md)
The Dual-Lobe Henyey-Greenstein Phase Function models light scattering in participating media by combining a forward-scattering lobe and a backward-scattering lobe to create a more realistic angular distribution than a single lobe. It sets itself apart from the parent [Multiple Octaves Phase Function](4939-multiple-octaves-phase-function.md) by specifically using a two-lobed structure rather than a general composite, providing a computationally efficient approximation of Mie scattering with significant back-scatter. It is not scientifically correct but is used for visual aesthetics.

### Weighted Sum of Phase Functions
A generalization for complex phase functions not described well by a single asymmetry parameter, modeled as a weighted sum of phase functions (like Henyey-Greenstein) with different parameters [S2]. The formula is $p(\omega_o, \omega'_o) = \sum_{i=1}^n w_i p_i(\omega_o \rightarrow \omega'_o)$, where $\sum w_i = 1$ [S2]. The pitfall is that the weights $w_i$ must sum to one to maintain normalization [S2]. This generalization is not provided in pbrt but is noted as easy to add [S2].

## Operating characteristics
- **Time Complexity:** $O(K)$ where $K$ is the number of octaves (4 in the standard implementation) [S1, unofficial]. Each step requires two Henyey-Greenstein evaluations [S1, unofficial].
- **Memory Complexity:** $O(1)$. No large data structures are required [S1, unofficial].
- **Performance:** Very fast due to the fixed number of iterations and simple analytical functions [S1, unofficial].
- **Strengths:** Capable of producing complex, multi-lobed scattering patterns (e.g., forward and backward scattering) with minimal computational overhead compared to tabulated or physical Mie scattering models [S1, unofficial].
- **Limitations:** The specific implementation does not guarantee a normalized phase function, which can lead to incorrect energy conservation if not handled carefully in the renderer [S1, unofficial]. It is an empirical fit rather than a physically derived model [S1, unofficial].
- **When to use:** When a simple, fast approximation of complex scattering (like cloud scattering) is needed and the overhead of more complex phase functions is unjustified [S1, unofficial].
- **When not to use:** When strict energy conservation and physical accuracy are required, or when the scattering distribution is known to require more than 4 octaves to model accurately [S1, unofficial].

## Background
The Multiple Octaves Phase Function originates from the `volumetrics` project by al-ro [S1, unofficial]. It is an empirical technique used in real-time or interactive rendering contexts to approximate complex scattering behavior [S1, unofficial]. The approach of summing multiple Henyey-Greenstein functions is a known generalization in volume rendering literature, as noted in the pbr-book [S2], but the specific "4-octave" formulation with the defined decay factors is unique to the source [S1, unofficial].

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
[S2] 11.2 Phase Functions - pbr-book.org, https://pbr-book.org/3ed-2018/Volume_Scattering/Phase_Functions, fetched 2026-10-27
[S3] Not given in the sources.
