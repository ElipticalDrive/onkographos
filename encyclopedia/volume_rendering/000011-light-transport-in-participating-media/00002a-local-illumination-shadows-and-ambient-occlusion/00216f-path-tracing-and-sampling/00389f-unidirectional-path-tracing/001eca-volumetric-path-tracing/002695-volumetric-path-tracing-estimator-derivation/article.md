# Volumetric Path Tracing Estimator Derivation

Also called: Generalized Path Contribution, Generalized Path Contribution Estimator, Monte Carlo Estimator for Volumetric Path Tracing

## Overview
This article derives the unbiased Monte Carlo estimator for the generalized path contribution function $\hat{P}$ in volumetric path tracing [S1]. It solves the problem of computing the expected radiance contribution of an $n$-vertex path through participating media by simplifying the general path integral formulation [S2, book]. It sets itself apart from [Volumetric Path Tracing](1863-volumetric-path-tracing.md) by providing the rigorous mathematical foundation for the estimator used in that method, specifically showing how distance sampling, event type selection, and direction sampling probabilities cancel out complex transmittance and geometric terms to yield a simplified product of phase functions and emission [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   An $n$-vertex path $\bar{p}_n = (p_1, \dots, p_n)$, where $p_1$ is the camera position and $p_n$ is the final vertex [S1].
*   $m$ real-scattering vertices within the path (excluding absorption and null scattering events) [S1].
*   The generalized scattering distribution function $\hat{f}$ [S2, book].
*   The path space emission function $\hat{L}_e$ [S1].
*   Sampling probabilities for distance, event type, and direction [S1].

**Outputs:**
*   The estimated path contribution $\hat{P}(\bar{p}_n)$ [S1].

### Data Structures
The method operates on the path vertices $p_i$ and associated directions $\omega_i$ [S1]. No specific complex data structures are required beyond standard vector/matrix types for positions, directions, and normals [G]. The key logical structure is the sequence of vertices categorized by their scattering event type (real scattering, null scattering, or absorption) [S1].

### Definitions and Notation
*   $\bar{p}_n$: A path with $n$ vertices [S1].
*   $p_i$: The position of the $i$-th vertex [S1].
*   $\omega_i$: The direction from $p_i$ to $p_{i+1}$ [S1].
*   $V$: The volume domain [S2, book].
*   $A$: The surface domain [S2, book].
*   $\sigma_{maj}$: The majorant extinction coefficient used for delta tracking [S1].
*   $\sigma_a, \sigma_s, \sigma_n$: Absorption, scattering, and null scattering coefficients, respectively [S1].
*   $\hat{f}(p_{i+1} \to p_i \to p_{i-1})$: The generalized scattering distribution function at vertex $p_i$ [S2, book].
*   $\hat{G}(p \leftrightarrow p')$: The generalized geometric term between points $p$ and $p'$ [S2, book].
*   $T_r(p \to p')$: The transmittance between $p$ and $p'$ [S2, book].
*   $C_p(p, p')$: The cosine factor for vertex $p$ in the geometric term [S2, book].
*   $p_{maj}(p_{i+1}|p_i, \omega_i)$: Probability density of sampling point $p_{i+1}$ along direction $\omega_i$ from $p_i$ [S1].
*   $p_e(p_i)$: Discrete probability of sampling the scattering event type at $p_i$ [S1].
*   $p_\omega(\omega_{i+1}|r_i, \omega_i)$: Probability density of sampling direction $\omega_{i+1}$ after a real scattering event at $r_i$ with incident direction $\omega_i$ [S1].
*   $p(r_{i-1} \to r_i \to r_{i+1})$: The phase function value for real scattering at $r_i$ [S1].

### Step-by-Step Derivation

1.  **General Estimator Formulation**
    The generalized path contribution is defined as the integral over the path space measure $\mu_{n-1}$:
    $$ \hat{P}(\bar{p}_n) = \int_{\mathcal{P}_{n-1}} L_e(p_n \to p_{n-1}) \hat{T}(\bar{p}_n) d\mu_{n-1}(p_2, \dots, p_n) $$ [S2, book].
    The Monte Carlo estimator for a sampled path $\bar{p}_n$ is:
    $$ \hat{P}(\bar{p}_n) = \frac{\hat{T}(\bar{p}_n) \hat{L}_e(p_n \to p_{n-1})}{\left(\prod_{i=0}^{n-1} p_{maj}(p_{i+1}|p_i, \omega_i)\right) \left(\prod_{i=1}^{n} p_e(p_i)\right) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i) \hat{G}(r_i \leftrightarrow r_{i+1})\right)} $$ [S1].
    Here, $\hat{T}(\bar{p}_n)$ is the path throughput function:
    $$ \hat{T}(\bar{p}_n) = \prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1}) \hat{G}(p_{i+1} \leftrightarrow p_i) $$ [S2, book].
    The generalized geometric term is:
    $$ \hat{G}(p \leftrightarrow p') = V(p \leftrightarrow p') T_r(p \to p') \frac{C_p(p, p') C_{p'}(p', p)}{\|p - p'\|} $$ [S2, book].
    where $V$ is the visibility function, and $C_p$ is the cosine factor:
    $$ C_p(p, p') = \begin{cases} |\mathbf{n}_p \cdot \frac{p - p'}{\|p - p'\|}| & \text{if } p \text{ is a surface vertex} \\ 1 & \text{otherwise} \end{cases} $$ [S2, book].
    The generalized scattering distribution function is:
    $$ \hat{f}(p_{i+1} \to p_i \to p_{i-1}) = \begin{cases} \sigma_s p(p_{i+1} \to p_i \to p_{i-1}) & \text{if } p_i \in V \\ f(p_{i+1} \to p_i \to p_{i-1}) & \text{if } p_i \in A \end{cases} $$ [S2, book].

2.  **Distance Sampling Simplification**
    Assume a single majorant $\sigma_{maj}$ is used for delta tracking [S1]. The probability density for sampling the distance $t$ to the next interaction is:
    $$ p_{maj}(p_{i+1}|p_i, \omega_i) = \sigma_{maj} e^{-\sigma_{maj} t} $$ [S1].
    The term $\hat{T}$ contains transmittance factors $T_{maj}$ derived from the majorant [S1]. When expanding $\hat{T}$ and dividing by the product of $p_{maj}$ terms in the denominator, the exponential factors $e^{-\sigma_{maj} t}$ cancel out the $T_{maj}$ factors [S1]. Each vertex contributes a factor of $1/\sigma_{maj}$ [S1]. Additionally, the $\hat{G}$ factors in the numerator (from $\hat{T}$) cancel with the $\hat{G}$ factors in the denominator (from the path space measure conversion) [S1]. This leaves a factor of $(\sigma_{maj})^n$ in the denominator:
    $$ \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1})\right) \hat{L}_e(p_n \to p_{n-1})}{\left(\sigma_{maj}\right)^n \left(\prod_{i=1}^{n} p_e(p_i)\right) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)} $$ [S1].

3.  **Event Type Simplification**
    The discrete probability $p_e(p_i)$ of choosing a specific event type (absorption, real scattering, or null scattering) at vertex $p_i$ is proportional to the corresponding coefficient divided by the majorant:
    $$ p_e(p_i) = \frac{\sigma_{\{a,s,n\}}(p_i)}{\sigma_{maj}} $$ [S1].
    The product $\prod_{i=1}^{n} p_e(p_i)$ therefore contains a factor of $(\sigma_{maj})^{-n}$ [S1]. This cancels the $(\sigma_{maj})^n$ term in the denominator from the previous step [S1]. The estimator becomes:
    $$ \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1})\right) \hat{L}_e(p_n \to p_{n-1})}{\left(\prod_{i=1}^{n} \sigma_{\{a,s,n\}}(p_i)\right) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)} $$ [S1].

4.  **Specific Event Constraints**
    In a standard path tracing setup, the first $n-1$ vertices are sampled as either real scattering or null scattering events (the path continues) [S1]. The last vertex $p_n$ is sampled as an absorption event (the path terminates) [S1]. Thus:
    *   For $i = 1 \dots n-1$, $\sigma_{\{a,s,n\}}(p_i)$ is either $\sigma_s(p_i)$ or $\sigma_n(p_i)$ [S1].
    *   For $i = n$, $\sigma_{\{a,s,n\}}(p_n)$ is $\sigma_a(p_n)$ [S1].
    The estimator simplifies to:
    $$ \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1})\right) \hat{L}_e(p_n \to p_{n-1})}{\left(\prod_{i=1}^{n-1} \sigma_{\{s,n\}}(p_i)\right) \sigma_a(p_n) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)} $$ [S1].

5.  **Phase Function and Null Scattering Cancellation**
    Neglecting surface scattering, $\hat{f}$ represents either regular volumetric scattering or null scattering [S1].
    *   For real scattering at $r_i$, $\hat{f} = \sigma_s p(r_{i-1} \to r_i \to r_{i+1})$ [S1].
    *   For null scattering at $n_i$, $\hat{f} = \sigma_n \cdot 1$ (the Heaviside function for null scattering is 1) [S1].
    The term $\sigma_{\{s,n\}}(p_i)$ in the denominator matches the $\sigma_s$ or $\sigma_n$ factor in the numerator's $\hat{f}$ [S1]. These cancel out for all $i=1 \dots n-1$ [S1].
    We restrict the product in the numerator to only the $m$ real-scattering vertices $r_1, \dots, r_m$ [S1]. The null scattering events contribute a factor of 1 and cancel their own $\sigma_n$ terms [S1]. The estimator becomes:
    $$ \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{m-1} p(r_{i-1} \to r_i \to r_{i+1})\right) \hat{L}_e(p_n \to p_{n-1})}{\sigma_a(p_n) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)} $$ [S1].
    Note: The product index for the phase functions corresponds to the real scattering vertices. If $m=0$, the product is empty (value 1) [G].

6.  **Final Emission Cancellation**
    The path space emission function $\hat{L}_e(p_n \to p_{n-1})$ includes a factor of $\sigma_a(p_n)$ (the probability of absorption at the last vertex) [S1]. This $\sigma_a(p_n)$ cancels with the $\sigma_a(p_n)$ in the denominator [S1].
    The final simplified estimator is:
    $$ \hat{P}(\bar{p}_n) = \left(\prod_{i=1}^{m-1} \frac{p(r_{i-1} \to r_i \to r_{i+1})}{p_\omega(\omega_{i+1}|r_i, \omega_i)}\right) \hat{L}_e(p_n \to p_{n-1}) $$ [S1].
    This result shows that the path contribution is the emission at the last vertex scaled by the product of ratios of phase function values to sampling probabilities for all real scattering events [S1].

### Pseudocode
The following Python-like pseudocode computes the estimator for a given path [S1]. It assumes the path vertices have been sampled and categorized [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_path_estimator(phase_functions, sampling_probs, emission_at_last_vertex):
    """
    Computes the simplified volumetric path contribution estimator.
    
    Args:
        phase_functions: List of phase function values p(r_{i-1} -> r_i -> r_{i+1}) for each real scattering vertex r_i.
        sampling_probs: List of direction sampling probabilities p_omega(omega_{i+1} | r_i, omega_i) for each real scattering vertex r_i.
        emission_at_last_vertex: The value of L_e(p_n -> p_{n-1}) including the sigma_a factor.
        
    Returns:
        The estimated path contribution P_hat.
    """
    m = len(phase_functions)
    if m == 0:
        # No real scattering events. The product is empty (1).
        # The estimator is just the emission at the last vertex.
        # Note: emission_at_last_vertex already includes sigma_a(p_n) which cancels the denominator sigma_a.
        return emission_at_last_vertex
    
    product_ratio = 1.0
    # The derivation specifies the product runs from i=1 to m-1 [S1].
    # In 0-based indexing, this corresponds to indices 1 through m-1.
    for i in range(1, m):
        product_ratio *= phase_functions[i] / sampling_probs[i]
        
    return product_ratio * emission_at_last_vertex
```

### Parameters and Edge Cases
*   **Majorant $\sigma_{maj}$**: Must be greater than or equal to the maximum extinction coefficient in the volume [S1]. If not, delta tracking is invalid [S1].
*   **Null Scattering**: The Heaviside function for null scattering is always 1 [S1]. Null scattering events do not contribute to the phase function product but consume a vertex in the path [S1].
*   **Surface Vertices**: The derivation assumes scenes with only volumetric scattering for simplicity [S1]. If surface vertices are present, $\hat{f}$ falls back to the BSDF $f$, and $C_p$ incorporates the cosine [S2, book]. The cancellation of $\sigma_{maj}$ and $p_e$ still holds, but the geometric term $\hat{G}$ does not fully cancel if surface normals are involved [S2, book].
*   **Zero Probability**: If $p_\omega$ is zero for a sampled direction, the estimator is undefined (division by zero) [G]. This is prevented by ensuring the sampling distribution supports the sampled directions [G].

## Operating characteristics

### Time and Memory Complexity
*   **Time**: $O(m)$, where $m$ is the number of real scattering events in the path [G]. The estimator requires a single pass over the real scattering vertices to compute the product of ratios [G].
*   **Memory**: $O(1)$ auxiliary memory per path, assuming the path vertices and probabilities are available or computed on the fly [G].

### Measured Performance
Not given in the sources.

### Scaling
The estimator scales linearly with the number of scattering events [G]. It is independent of the majorant value $\sigma_{maj}$ after simplification, though $\sigma_{maj}$ affects the efficiency of the distance sampling (delta tracking) [S1].

### Strengths and Limitations
*   **Strengths**: The simplified estimator is unbiased and computationally efficient [S1]. It removes the need to explicitly compute transmittance and geometric terms for each segment, relying instead on the sampling probabilities [S1].
*   **Limitations**: It assumes the path was sampled using the specific three-step process (distance, event type, direction) [S1]. It does not handle light sampling or multiple scattering strategies directly (see [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md)) [S1].

### When to Use
Use this estimator when implementing a basic volumetric path tracer that uses delta tracking and uniform or phase-function-based direction sampling [G]. It is the foundation for debugging and ground-truth generation [S1].

## Background
The derivation is based on the path integral formulation of light transport in participating media [S2, book]. It originates from the work on volumetric path tracing, specifically detailed in [S1] and [S2, book]. The method simplifies the general path contribution estimator by exploiting the properties of delta tracking and the structure of the sampling probabilities [S1].

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
[S2] Light Transport II: Volume Rendering / The Equation of Transfer, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27 (book)
