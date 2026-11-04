# Evans' Normal Modification

Also called: Ambient Occlusion with Normal Modification, Evans Normal Modification AO, Normal perturbation AO

## Overview
Evans' Normal Modification is a variant of ambient occlusion that perturbs the surface normal at each sample step to better approximate sky visibility effects [S1, unofficial][S2, unofficial]. It differs from its sibling [Quilez Ambient Occlusion](1117-quilez-ambient-occlusion.md) by modifying the ray direction per step and changing the accumulation base from 2 to 1.1 [S1, unofficial][S2, unofficial]. It builds on [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md) by tilting the marching direction toward the sky dome [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface point $x$, initial normal $n$, scene distance field function $f(r)$, upward unit vector $u$, maximum steps $k_{max}$, step size $\Delta t$, scale parameter $s$, and normal modification parameter $\alpha$ [S1, unofficial][S2, unofficial].
*   **Output:** Final occlusion value in $[0, 1]$ [S1, unofficial][S2, unofficial].

**Data Structures:**
*   $k$: Integer loop counter [S1, unofficial][S2, unofficial].
*   $d$: Float distance query result [S1, unofficial][S2, unofficial].
*   $occlusion$: Float accumulator [S1, unofficial][S2, unofficial].
*   $r$: Float3 sample position [S1, unofficial][S2, unofficial].
*   $n$: Float3 current normal vector [S1, unofficial][S2, unofficial].

**Numbered Steps:**
1. Initialize $k = 1$, $d = 0$, $occlusion = 0$ [S1, unofficial][S2, unofficial].
2. Loop while $k < k_{max}$:
   a. Calculate sample position: $r = x + \Delta t \cdot k \cdot n$ [S1, unofficial][S2, unofficial].
   b. Query distance: $d = f(r)$ [S1, unofficial][S2, unofficial].
   c. Update occlusion: $occlusion = 1.1^k \cdot (k \cdot \|n\| \cdot \Delta t_k - d)$ [S1, unofficial][S2, unofficial].
   d. Modify normal: $n = n + \alpha u$ [S1, unofficial][S2, unofficial].
   e. Increment counter: $k = k + 1$ [S1, unofficial][S2, unofficial].
3. Return $1 - \text{clamp}(s \cdot occlusion, 0, 1)$ [S1, unofficial][S2, unofficial].

**Formulas:**
*   $r = x + \Delta t \cdot k \cdot n$: Sample position along the current normal, scaled by step index $k$ and step size $\Delta t$ [S1, unofficial][S2, unofficial].
*   $occlusion = 1.1^k \cdot (k \cdot \|n\| \cdot \Delta t_k - d)$: Occlusion estimate per step using base 1.1 instead of 2, where $\Delta t_k$ is the step size at iteration $k$ (typically constant $\Delta t$) and $\|n\|$ is the Euclidean length of the current normal [S1, unofficial][S2, unofficial].
*   $n = n + \alpha u$: Normal perturbation formula, where $u$ is the upward unit vector and $\alpha$ scales the tilt [S1, unofficial][S2, unofficial].
*   $1 - \text{clamp}(s \cdot occlusion, 0, 1)$: Final occlusion mapping to $[0, 1]$ using scale $s$ [S1, unofficial][S2, unofficial].

**Parameters:**
*   $\alpha$: Normal modification strength. Evans used $0.5$ [S1, unofficial][S2, unofficial].
*   $u$: Upward pointing unit vector of the scene (e.g., $(0, 1, 0)$) [S1, unofficial][S2, unofficial].
*   $k_{max}$: Maximum number of sample steps [S1, unofficial][S2, unofficial].
*   $\Delta t$: Step size for ray marching [S1, unofficial][S2, unofficial].
*   $s$: Scale variable determining darkening intensity [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Robustness:** The method is not robust; parameters lack obvious defaults and require scene-specific tuning [S1, unofficial][S2, unofficial].
*   **Exaggerated Occlusion:** Applying the algorithm globally can cause exaggerated occlusion or incorrect highlighting (e.g., a box appearing highlighted near its top) when shapes are near each other, requiring different parameters than those for isolated shapes like spheres [S1, unofficial][S2, unofficial].
*   **Hybrid Usage:** The method may need to be combined with the standard Evans AO algorithm; for instance, Algorithm 3.4 handles ground points while Algorithm 3.7 handles sphere points to avoid incorrect shading interactions [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evans_normal_modification_ao(x, n, f, u, k_max, delta_t, s, alpha):
    k = 1
    occlusion = 0.0
    current_n = n  # Work on a copy to avoid mutating the original if needed
    
    while k < k_max:
        # Calculate sample point position along the current normal
        r = x + delta_t * k * current_n
        
        # Query distance field
        d = f(r)
        
        # Update occlusion using the modified base 1.1
        # delta_t_k is the step size at iteration k (constant delta_t here)
        norm_n = math.sqrt(current_n[0]**2 + current_n[1]**2 + current_n[2]**2)
        term = k * norm_n * delta_t - d
        occlusion = (1.1 ** k) * term
        
        # Modify the normal vector to tilt towards the sky
        current_n = current_n + alpha * u
        
        k = k + 1
        
    # Final result
    result = 1.0 - max(0.0, min(1.0, s * occlusion))
    return result
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sample steps $k_{max}$, as each step performs a distance field query and vector arithmetic [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Constant $O(1)$ auxiliary memory, maintaining only scalar and vector variables [S1, unofficial][S2, unofficial].
*   **Strengths:** Better approximates sky visibility effects for curved surfaces, making sphere-traced images resemble reference images more closely than standard ray marching AO [S1, unofficial][S2, unofficial].
*   **Limitations:** Not robust across different geometries; requires manual tuning of parameters ($\alpha$, $s$, $k_{max}$) for different scene types [S1, unofficial][S2, unofficial].
*   **When to use:** When rendering isolated curved surfaces (e.g., spheres) where sky visibility is prioritized [S1, unofficial][S2, unofficial].
*   **When not to use:** For ground planes or box-like structures where standard AO performs better, or when a robust, parameter-free solution is required [S1, unofficial][S2, unofficial].

## Background
*   **Origin:** Based on work by Evans [Eva06a], who suggested influencing the surface point normal to better approximate sky effects [S1, unofficial][S2, unofficial].
*   **History:** Presented as Algorithm 3.7 in the source material, modifying Algorithm 3.4 by adding the normal modification step, changing the accumulation base from 2 to 1.1, and explicitly showing the ray equation [S1, unofficial][S2, unofficial].
*   **Usage:** Used in GPU ray marching of distance fields for ambient occlusion, particularly where visual fidelity for curved surfaces is prioritized over cross-geometry robustness [S1, unofficial][S2, unofficial].

## Sources
[S1] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
