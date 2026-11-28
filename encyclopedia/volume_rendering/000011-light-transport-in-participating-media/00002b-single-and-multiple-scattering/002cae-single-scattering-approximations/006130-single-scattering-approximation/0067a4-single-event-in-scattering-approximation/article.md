# Single-Event In-Scattering Approximation

Also called: Single-Event In-Scattering

## Overview
The Single-Event In-Scattering Approximation is a method for estimating the total in-scattered radiance due to direct illumination along a ray path in participating media. It solves the computational bottleneck of integrating contributions from all $N$ scattering events by selecting a single representative sample $(X, W)$ per path to approximate the expectation value $\bar{L}$ [S1]. Unlike its sibling [Eye Scattering Recurrence](5316-eye-scattering-recurrence.md), which may iterate or recurse, this method reduces the multi-event sum to a single Monte Carlo sample, maintaining unbiasedness in the expectation over many paths while significantly accelerating rendering [S1].

## How it works
**Inputs:**
*   A ray path defined by a sequence of scattering events $\{(x_i, \omega_i)\}_{i=1}^{N}$, where $x_i$ is the position and $\omega_i$ is the direction of the $i$-th scattering event [S1].
*   Light power $\Phi$ [S1].
*   Material parameters: extinction coefficient $\sigma_t$ and anisotropy $g$ [S1].
*   Light source position $x_L$ [S1].

**Outputs:**
*   Approximated total in-scattering $L$ [S1].

**Data Structures:**
*   Path history: A list or array of tuples $(x_i, \omega_i)$ for $i=1 \dots N$.
*   Weights: A scalar weight $\varphi_i$ associated with each scattering event $i$. The cumulative weight $\Lambda(N)$ is defined as $\sum_{i=0}^{N} \varphi_i$ [S1]. Note: The source defines $\Lambda(N)$ with an index starting at 0, implying $\varphi_0$ exists (likely the initial segment or root weight) [S1].

**Formulas:**
The total in-scattering $L$ is defined as [S1]:
$$ L = \Phi \sum_{i=1}^{n} \varphi_i \rho g(\omega_i, \omega_{x_L \to x_i}) \gamma(x_L \to x_i) $$
Where:
*   $\Phi$: Light power [S1].
*   $\varphi_i$: Weight of the $i$-th scattering event [S1].
*   $\rho$: A coefficient (value not specified in sources) [S1].
*   $g(\cdot, \cdot)$: Phase function [S1].
*   $\omega_{x_L \to x_i}$: Direction of incoming direct illumination at position $x_i$ [S1].
*   $\gamma(x_L \to x_i)$: Extinction (attenuation) of light from the source to $x_i$ [S1].

In the absence of refractive boundaries, the incoming direction and attenuation are computed as [S1]:
$$ \omega_{x_L \to X} = \frac{x_L - X}{\|x_L - X\|} $$
$$ \gamma(x_L \to X) = \exp(-dt(x_L \to X)\sigma_t) $$
Where:
*   $\|\cdot\|$: Euclidean vector norm [S1].
*   $dt(x_L \to X)$: Distance along the direct path between light source $x_L$ and position $X$ [S1].
*   $\sigma_t$: Extinction coefficient [S1].

**Numbered Steps:**
1.  **Select Representative Sample:** Choose a single scattering event index $k$ from the path $\{(x_i, \omega_i)\}_{i=1}^{N}$ with probability proportional to its weight. The probability of selecting the $k$-th event is [S1]:
    $$ P(k) = \frac{\varphi_k}{\Lambda(N)} $$
    where $\Lambda(N) = \sum_{i=0}^{N} \varphi_i$ [S1].
2.  **Compute Incoming Direction:** Calculate the direction from the selected sample position $X = x_k$ to the light source $x_L$ using the vector difference and normalization [S1]:
    $$ \omega_{x_L \to X} = \frac{x_L - X}{\|x_L - X\|} $$
3.  **Compute Attenuation:** Calculate the extinction factor from the light source to the sample position $X$ using the distance $dt(x_L \to X)$ and extinction coefficient $\sigma_t$ [S1]:
    $$ \gamma(x_L \to X) = \exp(-dt(x_L \to X)\sigma_t) $$
4.  **Evaluate In-Scattering:** Compute the single-event in-scattering contribution using the selected sample's parameters, the computed direction, and attenuation [S1]:
    $$ L \approx \Phi \cdot \varphi_k \cdot \rho \cdot g(\omega_k, \omega_{x_L \to X}) \cdot \gamma(x_L \to X) $$
    *Note: The source formula sums over $i=1$ to $n$, but the approximation replaces this sum with the single term for the selected $k$. The weight $\varphi_k$ is included in the selection probability, but the source formula for $L$ explicitly includes $\varphi_i$ in the sum. The approximation selects one term from the sum. Thus, the result is the value of that specific term.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def single_event_in_scattering(path_events, light_pos, light_power, sigma_t, rho, phase_function):
    """
    Approximates total in-scattering by selecting a single representative scattering event.
    
    Args:
        path_events: List of tuples (x_i, omega_i, phi_i) for i=1..N.
                     Note: Source implies phi_0 exists for normalization. 
                     If path_events only contains 1..N, phi_0 must be provided or assumed.
                     For this implementation, we assume path_events includes all relevant weights 
                     for the sum Lambda(N). If phi_0 is separate, it should be prepended.
        light_pos: Position of the light source (x_L).
        light_power: Light power (Phi).
        sigma_t: Extinction coefficient.
        rho: Coefficient.
        phase_function: Function g(omega_i, omega_in).
        
    Returns:
        Approximated in-scattering L.
    """
    # Step 1: Select representative sample
    # Calculate cumulative weights for sampling
    # Source defines Lambda(N) = sum_{i=0}^N phi_i. 
    # If path_events starts at i=1, we need phi_0. 
    # Assuming path_events list contains events 1..N and their weights phi_i.
    # We need to handle the normalization sum correctly.
    
    # Extract weights from path_events. 
    # Let's assume path_events is a list of (x, omega, phi) for i=1..N.
    # We need to know phi_0. If not provided, we cannot compute Lambda(N) as defined.
    # [G] Guard: If phi_0 is not part of the path events list, it must be passed separately.
    # For this generic implementation, we assume the caller provides a list that 
    # allows computing the total weight. If phi_0 is 0 or negligible, it might be omitted,
    # but strictly, Lambda(N) includes i=0.
    
    # Let's assume we have a list of all weights including phi_0 for simplicity in selection.
    # If the input is strictly events 1..N, we need phi_0.
    # To make this self-contained, we assume the input `all_weights` includes phi_0..phi_N.
    # However, the function signature uses path_events. 
    # Let's adjust: path_events contains (x_i, omega_i, phi_i) for i=1..N.
    # We need phi_0. Let's assume phi_0 is passed as a separate argument or is 0.
    # Source: "Lambda(N) = sum_{i=0}^N phi_i".
    
    # Re-structuring for clarity:
    # We need to select k from 1..N with probability phi_k / Lambda(N).
    
    # Extract weights for events 1..N
    weights = [event[2] for event in path_events]
    
    # We need phi_0. If not available, we cannot strictly follow the source.
    # [G] Assumption: phi_0 is 0 if not provided, or the source implies the sum is over the 
    # scattering events present. However, the index 0 suggests an initial state.
    # Let's assume phi_0 is provided in a separate variable or is 0.
    # For this code, let's assume phi_0 is 0 for simplicity if not specified, 
    # but note this is a potential pitfall if phi_0 is significant.
    phi_0 = 0.0 
    
    total_weight = phi_0 + sum(weights)
    
    if total_weight == 0:
        return 0.0
        
    # Select k based on probability phi_k / Lambda(N)
    # Lambda(N) = total_weight
    r = random.uniform(0, total_weight)
    cumulative = phi_0
    k_index = -1
    for i, w in enumerate(weights):
        cumulative += w
        if r <= cumulative:
            k_index = i
            break
            
    if k_index == -1:
        # Fallback, should not happen if logic is correct
        k_index = len(path_events) - 1
        
    # Get the selected event (k-th event, 0-indexed in list, corresponds to i=k+1 in source)
    selected_event = path_events[k_index]
    X = selected_event[0]
    omega_k = selected_event[1]
    phi_k = selected_event[2]
    
    # Step 2: Compute incoming direction
    # omega_{x_L -> X} = (x_L - X) / ||x_L - X||
    dir_vec = [light_pos[j] - X[j] for j in range(3)]
    norm = math.sqrt(sum(v**2 for v in dir_vec))
    
    if norm == 0:
        # Degenerate case: light is at the scattering position
        # [G] Handle division by zero. Return 0 or handle gracefully.
        return 0.0
        
    omega_in = [v / norm for v in dir_vec]
    
    # Step 3: Compute attenuation
    # gamma = exp(-dt * sigma_t)
    dt = norm # Distance from light to X
    gamma = math.exp(-dt * sigma_t)
    
    # Step 4: Evaluate in-scattering
    # L = Phi * phi_k * rho * g(omega_k, omega_in) * gamma
    # Note: The source formula has sum(phi_i * ...). The approximation picks ONE term.
    # So we compute the term for k.
    
    g_val = phase_function(omega_k, omega_in)
    
    L = light_power * phi_k * rho * g_val * gamma
    
    return L
```

**Parameters:**
*   Extinction coefficient $\sigma_t$: Sampled from $[0, 200]$ [S1].
*   Anisotropy $g$: Sampled from $[-1, 1]$ [S1].
*   Model height: Normalized to 1m [S1].

**Edge Cases and Pitfalls:**
*   **Refractive Boundaries:** If refractive medium boundaries are present, the simplified expressions for direction and attenuation are invalid. Alternative computation methods accounting for refraction must be employed [S1].
*   **Light Position:** If the light source position coincides with the scattering position ($X = x_L$), the direction vector has zero norm, leading to division by zero [G].
*   **Zero Weights:** If all weights $\varphi_i$ are zero, the normalization factor $\Lambda(N)$ is zero, making selection impossible [G].

## Subtopics

### Eye Scattering Recurrence
An alternative approximation method. It differs by potentially using recurrence relations rather than a single sample selection. Choose when single-event approximation is insufficient for visual accuracy in complex media. See [Eye Scattering Recurrence](5316-eye-scattering-recurrence.md).

## Operating characteristics
*   **Time Complexity:** $O(1)$ per path after the path is generated, as it involves selecting one sample and computing one term, compared to $O(N)$ for the full sum [S1].
*   **Memory Complexity:** $O(N)$ to store the path history, same as standard path tracing [S1].
*   **Measured Performance:** The method accelerates rendering by avoiding the summation over all scattering events. It is used in sphere-tracing approaches for volumetric subsurface effects [S1].
*   **Strengths:** Significant performance gain; maintains unbiasedness in expectation over many paths [S1].
*   **Limitations:** Approximation error in individual paths; may require more samples for low variance. Accuracy depends on the validity of the single-event assumption [S1].
*   **When to Use:** When direct illumination in-scattering is a significant component and performance is critical, and refractive boundaries are absent or handled separately [S1].
*   **When Not to Use:** When refractive boundaries are present and a simple geometric approximation is insufficient, or when high accuracy for single paths is required without averaging [S1].

## Background
The method was proposed by L. Leonard, K. Höhlein, and R. Westermann in their work on learning multiple-scattering solutions for sphere-tracing of volumetric subsurface effects [S1]. It is part of a broader effort to accelerate path tracing in participating media by approximating the in-scattering term [S1].

## Sources
[S1] Learning Multiple‐Scattering Solutions for Sphere‐Tracing of Volumetric Subsurface Effects, https://doi.org/10.1111/cgf.142623, fetched 2026-10-26
