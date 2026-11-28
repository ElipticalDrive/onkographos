# Adaptive Volume Rendering Integration

Also called: Adaptive Numerical Integration for Volume Rendering, Adaptive VRI evaluation

## Overview
Adaptive Volume Rendering Integration (VRI) is a method for computing the Volume Rendering Integral using adaptive numerical integration with step size control. It separates the integration into an inner integral (handling density/opacity variations) and an outer integral (handling color/transparency variations) to achieve high accuracy with fewer samples than uniform Riemann summation [S1]. It is distinguished from sibling methods like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by its explicit coupling of error estimation to the volume rendering equation's specific structure (inner/outer integrals) rather than generic geometric or density field properties [S1].

## How it works
**Inputs:**
- $t$: Current ray parameter (start of integration interval).
- $h$: Initial step size guess ($h^*_0$).
- $\epsilon$: Error tolerance.
- $h_{min}$: Minimum step size limit.
- $h_{max}$: Maximum step size limit.

**Outputs:**
- $I$: Accumulated Volume Rendering Integral (vector $[R, G, B, A]$).

**Data Structures:**
- The integral $I$ is a 4-component vector for RGBA channels.
- Step sizes $h_{inner}$ and $h_{outer}$ are scalar floats.

**Core Concepts:**
The method computes the VRI by iteratively advancing along the ray. At each step, it evaluates an **Inner Integral** ($RI$) to determine a safe step size and check for medium transparency. If the medium contributes, it evaluates an **Outer Integral** ($RO$) to compute the color/opacity contribution for that segment. The step size is adjusted dynamically based on an error estimate $E(t, h)$ [S1].

**Algorithms:**

1.  **Inner Integral Procedure ($RI$):**
    -   Computes the integral over a small interval to ensure accuracy within tolerance.
    -   Returns the integral value and the valid step size $h_{inner}$.
    -   Uses the adaptive loop (Algorithm 2) but terminates as soon as a valid step size is found.
    -   Ensures $h_{inner} \in [h_{min}, h_{max}]$.

2.  **Outer Integral Procedure ($RO$):**
    -   Computes the full VRI contribution (color/alpha) for the interval.
    -   Uses regular (non-adaptive) Simpson's rule for the inner integral evaluation within this step, as the error is already bounded by $h_{inner}$ [S1].
    -   Iterates until the whole ray segment is covered.
    -   Ensures $h_{outer} \in [h_{min}, h_{max}]$.

3.  **Adaptive Step Control (Algorithm 2):**
    This is the core loop used by both $RI$ and $RO$ to determine step sizes.
    -   Initialize: $I = 0$, $t_{final} = t + h$, $h = \min(h^*_0, t_{final} - t)$, $lastsucceeded = true$.
    -   Loop while $t < t_{final}$:
        -   Evaluate error $E(t, h)$.
        -   **If $E(t, h) \le \epsilon$ (Success):**
            -   Add integral $D(t, h) + E(t, h)$ to $I$. [Note: Source notation $D(t,h)+E(t,h)$ implies the integral value plus error term, or simply the accumulated integral value. Implementation should add the computed integral value for the step].
            -   Advance $t = t + h$.
            -   If $lastsucceeded$ was true:
                -   Double step size: $h = 2 \cdot h$.
                -   Double tolerance: $\epsilon = 2 \cdot \epsilon$.
            -   Else:
                -   Set $lastsucceeded = true$.
            -   Cap step: $h = \min(h, t_{final} - t)$.
        -   **If $E(t, h) > \epsilon$ (Failure):**
            -   Halve step size: $h = h / 2$.
            -   Halve tolerance: $\epsilon = \epsilon / 2$.
            -   Set $lastsucceeded = false$.
            -   Loop repeats with new $h$.
    -   Return $I$.

4.  **Main Integration Loop (Algorithm 3):**
    -   Initialize $I = [R=0, G=0, B=0, A=0]$.
    -   Set $t_{final} = t + h$.
    -   Set $h_{inner} = \min(h^*_0, t_{final} - t)$.
    -   Set $h_{outer} = h_{inner}$ (for initialization).
    -   Loop while $t < t_{final}$:
        -   Call $RI(t, t_{final} - t, h_{inner})$ to get $I_{inner}$ and $h_{inner}$.
        -   If $I_{inner} \neq 0$ (medium is not transparent):
            -   **Coupled Strategy:** Set $h_{outer} = h_{inner}$.
            -   **Decoupled Strategy:** $h_{outer}$ is controlled independently (initialized to $h_{inner}$ but evolves separately).
            -   Call $RO(t, h_{inner}, h_{outer})$ to get $I_{outer}$ and $h_{outer}$.
            -   Add $I_{outer}$ to $I$.
            -   Update $t$ based on the progress of the outer integration (implied by $RO$ returning integral and step info; effectively $t$ advances by the step taken in $RO$).
    -   Return $I$.

**Coupling Strategies:**
-   **Coupled:** The inner integral dictates the initial step size for the outer integral at each iteration. Assumes coherence between density and color variations. Reuses intermediate Simpson's values from the inner integral [S1].
-   **Decoupled:** The outer step is controlled independently. The inner step only limits the integration interval. Theoretically better for high-frequency transfer function variations [S1].

**Parameters:**
-   $h^*_0$: Initial step size. Default: 0.5 voxel units [S1].
-   $h_{min}$: Minimum step size. Default: 0.1 voxel units [S1]. Prevents infinite halving if error cannot be estimated precisely [S1].
-   $h_{max}$: Maximum step size. Default: 2.0 voxel units [S1]. Prevents missing high-frequency details (e.g., thin leaves) [S1].
-   $\epsilon$: Error tolerance. Tested values: 0.01, 0.005, 0.001 [S1].

**Edge Cases and Pitfalls:**
-   **Overshooting Features:** Without $h_{max}$, high-frequency variations (e.g., thin leaves) may be missed because the adaptive step size becomes too large [S1].
-   **Numerical Inaccuracy:** Error evaluation has inherent inaccuracies. Even with strict tolerance, some rays may exceed the limit due to these inaccuracies and the $h_{min}$ floor [S1].
-   **High-Frequency Ignorance:** High-frequency regions may be wrongly ignored if the error evaluation sampling strategy fails to detect them, especially if the initial step size is too large [S1].
-   **GPU vs CPU:** GPU SIMD architecture favors Riemann summation because adaptive approaches require different step sizes per thread, reducing parallel efficiency. CPU architecture favors adaptive algorithms [S1].

## Operating characteristics
-   **Accuracy:** Offers better accuracy for similar or lower computational cost compared to Riemann summation. Riemann summation requires prohibitive computational costs to reach similar accuracy with constant small step sizes [S1].
-   **Performance:**
    -   **CPU:** Favors adaptive algorithms. Average frame rendering times were lower than uniform sampling for equivalent or better accuracy [S1].
    -   **GPU:** Favors Riemann summation. Adaptive approaches reduce parallel efficiency due to divergent step sizes per thread [S1].
-   **Complexity:** Adaptive step size control reduces the number of samples needed in smooth regions while increasing them in complex regions. The exact complexity depends on the scene's frequency content and the chosen tolerance [S1].
-   **Strengths:** High accuracy, efficient sampling in smooth regions, handles varying density and color transfer functions [S1].
-   **Limitations:** Poor parallel efficiency on GPUs, potential for missing high-frequency details if $h_{max}$ is not set, numerical inaccuracies in error estimation [S1].
-   **When to use:** CPU-based rendering, scenes with varying density/color frequencies where accuracy is critical, or when computational resources allow for per-ray adaptive control.
-   **When not to use:** GPU-based rendering where parallel efficiency is paramount, or when uniform sampling is sufficient for the required quality.

## Background
-   **Origin:** Proposed in "Accurate Volume Rendering based on Adaptive Numerical Integration" [S1].
-   **Development:** The method builds on iterative integration techniques, adapting them for the specific structure of the Volume Rendering Integral. It introduces the concept of inner and outer integrals to decouple density and color evaluation, allowing for more efficient step size control [S1].
-   **Usage:** Tested on datasets from volvis.org (Bonsai, BluntFin, Teapot, Engine) [S1].

## Sources
[S1] Accurate Volume Rendering based on Adaptive Numerical Integration, https://lhf.impa.br/ftp/papers/sib2015.pdf, fetched 2026-10-29
