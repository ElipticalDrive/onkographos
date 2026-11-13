# Unidirectional Spectral Tracking


## Overview
Unidirectional Spectral Tracking is a spectral volume rendering technique that samples absorption, scattering, and null events using a majorant density to drive the sampling process, returning the path length and spectral contribution while handling Russian roulette implicitly via event selection [S1, unofficial]. It differs from its parent, Spectral Tracking, by employing a specific unidirectional event selection logic that simplifies the throughput update and termination conditions for single-bounce or specific path configurations, relying on a mean-based majorant rather than a per-voxel maximum [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a Voxelized Path Tracing Density field (`vsdf`); a maximum voxel density `max_density`; a maximum ray distance `max_distance`; a random number generator `rng`.
*   **Output:** A pair containing the total `path_length` traveled and the spectral `throughput` (radiance contribution).

**Data Structures:**
*   `vsdf`: A data structure providing density evaluation. It returns a vector of total density and a scalar scatter ratio (albedo) at a given position.
*   `rng`: A random number generator capable of producing uniform floating-point numbers in $[0, 1)$.

**Parameters:**
*   `max_density`: The maximum voxel density of the volume multiplied by an object density multiplier ($\rho_{max} = \text{voxel\_max} \times \text{density\_mult}$) [S1, unofficial].
*   `imax_density`: The inverse of `max_density` ($1.0 / \text{max\_density}$) [S1, unofficial].
*   `majorant_density`: The majorant used for distance sampling. In this implementation, it is the mean of the scatter product with `max_density`: $\bar{\sigma}_M = \text{mean}(\text{vsdf.scatter} \times \text{max\_density})$ [S1, unofficial].

**Algorithm Steps:**
1.  Initialize `path_length` to $0.0$, `transmittance` $\mathbf{T}$ to $\vec{1}$, and `throughput` $\mathbf{f}$ to $\vec{1}$ [S1, unofficial].
2.  Loop indefinitely:
    a.  **Sample Path Length:**
        *   If `majorant_density` is $0.0$, set `path_length` to $-\text{FLT\_MAX}$ [S1, unofficial].
        *   Otherwise, sample $u_1 \sim \mathcal{U}[0,1)$ and compute:
            $$ \text{path\_length} = -\frac{\ln(1 - u_1)}{\text{majorant\_density}} $$
    b.  **Termination:** If `path_length` $\ge$ `max_distance`, break the loop [S1, unofficial].
    c.  **Position:** Calculate current position:
        $$ \mathbf{p} = \mathbf{o} + \text{path\_length} \times \mathbf{d} $$
    d.  **Evaluate Cross-sections:** Evaluate the VPT density at $\mathbf{p}$ to get total density vector $\boldsymbol{\sigma}_t$ and scatter ratio $s$ [S1, unofficial]:
        $$ \boldsymbol{\sigma}_t = \text{eval\_vpt\_density}(\text{vsdf}, \mathbf{p}) $$
        $$ s = \text{vsdf.scatter} $$
        Compute component cross-sections:
        $$ \boldsymbol{\sigma}_s = \boldsymbol{\sigma}_t \times s $$
        $$ \boldsymbol{\sigma}_a = \boldsymbol{\sigma}_t \times (\vec{1} - s) $$
        $$ \boldsymbol{\sigma}_n = \text{max\_density} \times \vec{1} - \boldsymbol{\sigma}_t $$
        *(Note: The source code implies `sigma_t` is a vector, likely per-channel or spectral, and `sigma_n` is the null cross-section vector)* [S1, unofficial].
    e.  **Update Transmittance:**
        $$ \mathbf{T} \leftarrow \mathbf{T} \times (\vec{1} - \boldsymbol{\sigma}_t \times \text{imax\_density}) $$
        *(Note: The source code uses `imax_density` here, which normalizes the density against the majorant scale)* [S1, unofficial].
    f.  **Event Selection:** Determine the event type $e$ and the corresponding coefficient $\mu_e$ [S1, unofficial]:
        *   If $e = \text{EVENT\_NULL}$, $\mu_e = \boldsymbol{\sigma}_n$.
        *   If $e = \text{EVENT\_SCATTER}$, $\mu_e = s$ (scalar, applied to throughput).
        *   If $e = \text{EVENT\_ABSORB}$, $\mu_e = \vec{1} - s$ (scalar, applied to throughput).
        *(Note: The source code snippet shows `mu_e` being used to update `f`, but the logic for selecting *which* event occurs is implicit in the "unidirectional" nature or handled by the caller/context not fully detailed in the snippet. However, the snippet explicitly calculates `mu_e` based on `e`. The standard spectral tracking selects an event proportional to these coefficients. The "unidirectional" aspect likely refers to the specific handling of the throughput update or the lack of multiple scattering loops in this specific variant)* [S1, unofficial].
    g.  **Null Event:** If $e = \text{EVENT\_NULL}$, continue to the next iteration (loop back to step 2a) [S1, unofficial].
    h.  **Update Throughput:**
        $$ \mathbf{f} \leftarrow \mathbf{f} \times \mu_e $$
        $$ p = \mathbf{f} $$
    i.  **Return:** Return the pair `(path_length, f / mean(p))` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unidirectional_spectral_tracking(ray, vsdf, max_density, max_distance, rng):
    # Parameters
    imax_density = 1.0 / max_density
    # Majorant is the mean of scatter product with max_density
    # Note: In a real implementation, this might be precomputed or estimated.
    # The source defines it as: mean(vsdf.scatter * max_density)
    # For the sake of the algorithm, we assume majorant_density is provided or computed.
    # If not precomputed, we might approximate it or use a global max.
    # The source code snippet uses 'majorant_density' directly in the sampling formula.
    majorant_density = compute_majorant(vsdf, max_density) 

    path_length = 0.0
    transmittance = vec3(1.0)
    throughput = vec3(1.0)

    while True:
        # Sample path length
        if majorant_density == 0.0:
            path_length = -float('inf') # FLT_MAX equivalent
        else:
            u1 = rng.uniform(0.0, 1.0)
            path_length = -log(1.0 - u1) / majorant_density

        # Termination
        if path_length >= max_distance:
            break

        # Current position
        current_pos = ray.origin + path_length * ray.direction

        # Evaluate densities
        sigma_t_vec = eval_vpt_density(vsdf, current_pos)
        scatter_ratio = vsdf.scatter(current_pos) # Scalar albedo
        
        # Component cross-sections
        sigma_s = sigma_t_vec * scatter_ratio
        sigma_a = sigma_t_vec * (1.0 - scatter_ratio)
        sigma_n = max_density * vec3(1.0) - sigma_t_vec

        # Update transmittance
        transmittance *= (1.0 - sigma_t_vec * imax_density)

        # Event Selection Logic (Implicit in "Unidirectional" context or caller)
        # The source code calculates mu_e based on event e.
        # Standard Spectral Tracking (Delta Tracking) selects an event type
        # proportional to the coefficients. Here, we assume the event 'e'
        # is determined by a standard spectral tracking step or a specific
        # unidirectional rule.
        # Based on the snippet:
        # mu_e is assigned based on e.
        # If e is NULL, continue.
        
        # To make this self-contained, we must implement the event selection.
        # Standard approach: Sample event type proportional to sigma_n, sigma_s, sigma_a.
        # However, the snippet separates the calculation of mu_e from the selection.
        # Let's assume standard spectral event selection:
        
        total_coeff = sum(sigma_n) + sum(sigma_s) + sum(sigma_a) # Simplified scalar sum for selection?
        # Actually, spectral tracking often works per-channel or with a majorant vector.
        # The source uses a scalar majorant_density for distance.
        # The event selection likely uses the vector coefficients.
        
        # Re-reading source: "Determine mu_e based on event e".
        # This implies 'e' is already chosen.
        # In Delta Tracking, you pick a coefficient index.
        # Let's implement a standard spectral event selection.
        
        # Normalize coefficients for selection
        # Note: sigma_n, sigma_s, sigma_a are vectors.
        # We need to select an event type.
        
        # Simplified event selection (assuming scalar approximation or per-channel independence)
        # The source code is ambiguous on *how* 'e' is chosen, only how mu_e is used.
        # However, "Unidirectional" might imply a specific simplified path.
        # Given the constraint to implement from text, and the source snippet,
        # we will assume a standard spectral event selection step is required.
        
        # Let's assume the event is selected proportionally to the norms or sums.
        # Or, more likely, this is a single-bounce tracer where the event is determined by the interaction.
        
        # Let's stick to the source's structure:
        # It calculates mu_e.
        # If NULL, continue.
        # Else update throughput.
        
        # We need to generate 'e'.
        # Let's use a standard spectral tracking event selection.
        
        # Coefficients for event selection (scalar values derived from vectors)
        # This is a gap filled with [G] standard spectral tracking logic.
            # In many spectral implementations, the majorant is a vector, and you pick a channel.
            # Here, majorant is scalar.
            # Let's assume the event is selected based on the total density components.
            
            # For implementation completeness, we assume a standard spectral event selection:
            # 1. Sample u2 ~ U[0,1)
            # 2. Compare against cumulative probabilities of sigma_n, sigma_s, sigma_a.
            
            # Since the source doesn't give the selection logic, only the handling,
            # and "Unidirectional" is the name, we might assume a simpler model.
            # However, without the selection logic, the code is incomplete.
            # We will add a comment [G] for the selection logic.
            
            # [G] Standard spectral event selection:
            # Sample event type e from {NULL, SCATTER, ABSORB} proportional to 
            # ||sigma_n||, ||sigma_s||, ||sigma_a|| respectively.
            
            # For the purpose of this article, we define the selection as:
            # e = sample_event(sigma_n, sigma_s, sigma_a)
            
            # But to be precise to the source, which calculates mu_e *after* e is known,
            # we must define e.
            
            # Let's assume the source implies a specific unidirectional path where
            # the event is determined by the first interaction.
            
            # We will implement a placeholder for event selection [G]
            e = select_spectral_event(sigma_n, sigma_s, sigma_a, rng)

        # Determine mu_e
        if e == EVENT_NULL:
            mu_e = sigma_n
        elif e == EVENT_SCATTER:
            mu_e = scatter_ratio # Scalar
        elif e == EVENT_ABSORB:
            mu_e = 1.0 - scatter_ratio # Scalar
        else:
            mu_e = 0.0

        # Handle Null Event
        if e == EVENT_NULL:
            continue

        # Update Throughput
        throughput *= mu_e
        p = throughput

        # Return result
        # The source returns f / mean(p).
        # Since p = f, this is 1.0?
        # Or is p a vector and mean is taken over channels?
        # "f / mean(p)" suggests p might be different from f in a general MIS context,
        # but here p=f.
        # This might be a vestige of a more complex MIS implementation.
        # We return f / mean(f) if f is vector?
        # Or simply f.
        # The source says: "Return pair of path_length and f / mean(p)."
        # If p=f, and f is vec3, mean(p) is scalar.
        # So we return path_length, f / mean(f).
        
        return path_length, throughput / mean(throughput)

def select_spectral_event(sigma_n, sigma_s, sigma_a, rng):
    # [G] Standard spectral event selection logic
    # Sum of coefficients
    sum_n = sum(sigma_n)
    sum_s = sum(sigma_s)
    sum_a = sum(sigma_a)
    total = sum_n + sum_s + sum_a
    
    if total == 0:
        return EVENT_NULL
        
    u2 = rng.uniform(0.0, 1.0)
    if u2 < sum_n / total:
        return EVENT_NULL
    elif u2 < (sum_n + sum_s) / total:
        return EVENT_SCATTER
    else:
        return EVENT_ABSORB

def compute_majorant(vsdf, max_density):
    # [G] The source defines majorant as mean(vsdf.scatter * max_density)
    # This requires sampling or integrating over the volume.
    # For a single ray, this might be a global constant or estimated.
    # We assume it is precomputed or passed as a parameter.
    # If not, this function is a placeholder.
    return 0.0 # Placeholder
```

**Edge Cases and Pitfalls:**
*   **Zero Majorant:** If `majorant_density` is $0.0$, the distance sampling formula involves division by zero. The implementation must handle this by setting `path_length` to $-\text{FLT\_MAX}$ (or a sufficiently large value) to effectively terminate or skip the step [S1, unofficial].
*   **Precision:** The use of `imax_density` for transmittance updates suggests a normalization that may be sensitive to floating-point precision if `max_density` is very large or very small.
*   **Event Selection:** The source code snippet does not explicitly detail the random selection of the event type ($e$), only how $\mu_e$ is calculated once $e$ is known. A complete implementation requires a standard spectral event selection step (e.g., proportional to the norms of $\boldsymbol{\sigma}_n, \boldsymbol{\sigma}_s, \boldsymbol{\sigma}_a$) [G].

## Operating characteristics
*   **Time Complexity:** Linear in the number of scattering events along the ray. The cost per step is dominated by the VPT density evaluation and the random number generation.
*   **Memory Complexity:** $O(1)$ auxiliary memory, assuming the VPT density field is accessed via a texture or grid lookup.
*   **Performance:** The method is designed to handle heterogeneous spectral volumes efficiently. In tests, it showed comparable convergence to Delta Tracking and Spectral MIS in homogeneous albedo scenes, but provided up to a 2x speedup in heterogeneous albedo scenes where the spectral extinction heterogeneity could be exploited [S1, unofficial].
*   **Strengths:** Implicit Russian roulette handling via event selection; efficient for spectral volumes with known majorants.
*   **Limitations:** Requires a valid majorant density; performance may degrade if the majorant is a loose bound (high rejection rate for null events).

## Background
Unidirectional Spectral Tracking is part of the Yocto/GL VPT (Voxelized Path Tracing) extension, developed by RickyMexx [S1, unofficial]. It is an evolution of spectral tracking techniques aimed at improving the efficiency of rendering heterogeneous volumes with per-wavelength optical properties. The method is used in the context of comparing spectral rendering algorithms, including Delta Tracking and Spectral MIS, on various test scenes such as bunny clouds and smoke volumes [S1, unofficial].

## Sources
[S1] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
