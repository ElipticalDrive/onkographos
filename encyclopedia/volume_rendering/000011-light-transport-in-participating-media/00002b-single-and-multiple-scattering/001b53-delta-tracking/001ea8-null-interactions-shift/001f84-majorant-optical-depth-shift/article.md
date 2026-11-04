# Majorant optical depth shift

Also called: majorant optical depth shift derivation, shift mapping of medium interaction depths

## Overview
The majorant optical depth shift maps a medium interaction depth $t$ sampled on a base viewing segment to a corresponding depth $t'$ on a target viewing segment. This enables path reuse in multi-view rendering by accounting for differences in volume geometry and extinction coefficients between views. It builds on [Null Interactions Shift](1836-null-interactions-shift.md) by providing the specific mapping function required when the medium's majorant extinction coefficients differ between the base and target segments.

## How it works
The method relies on the piecewise constant approximation of the majorant optical depth along a ray. It assumes that the ray passes through a sequence of volume sections, each with a constant majorant extinction coefficient. The mapping is derived by equating the scaled majorant optical depth on the base segment to the majorant optical depth on the target segment.

### Inputs and Outputs
- **Inputs**:
  - $t$: The interaction depth on the base segment.
  - Base segment data: A list of $N$ volume sections. Each section $j$ has entry depth $t_{\min, j}$, exit depth $t_{\max, j}$, and constant majorant extinction coefficient $\bar{\mu}_j$.
  - Target segment data: A list of $M$ volume sections. Each section $i$ has entry depth $t'_{\min, i}$, exit depth $t'_{\max, i}$, and constant majorant extinction coefficient $\bar{\mu}'_i$.
  - $\bar{\tau}_{\max}$: The total majorant optical depth of the base segment (sum of $\bar{\mu}_j (t_{\max, j} - t_{\min, j})$ for all $j$).
  - $\bar{\tau}'_{\max}$: The total majorant optical depth of the target segment (sum of $\bar{\mu}'_i (t'_{\max, i} - t'_{\min, i})$ for all $i$).
- **Output**:
  - $t'$: The mapped interaction depth on the target segment.

### Data Structures
The volume sections are typically stored as arrays or lists of structs/tuples.
- **Base Section $j$**:
  - `t_min`: float, entry depth.
  - `t_max`: float, exit depth.
  - `mu_bar`: float, majorant extinction coefficient.
- **Target Section $i$**:
  - `t_min`: float, entry depth.
  - `t_max`: float, exit depth.
  - `mu_bar`: float, majorant extinction coefficient.

### Parameters
- $C = \bar{\tau}'_{\max} / \bar{\tau}_{\max}$: The scaling constant relating the total majorant optical depths of the target and base segments.

### Steps
1. **Calculate the scaling constant**:
   Compute $C = \bar{\tau}'_{\max} / \bar{\tau}_{\max}$.

2. **Identify the containing sections**:
   Find the index $n_l$ such that $t_{\min, n_l} \le t < t_{\max, n_l}$ in the base segment.
   Find the index $n_{l'}$ such that $t'_{\min, n_{l'}} \le t' < t'_{\max, n_{l'}}$ in the target segment.
   *Note: $t'$ is initially unknown. The algorithm typically involves solving for $t'$ or iterating. However, the derivation in [S1] presents a closed-form solution assuming the sections are known. In practice, one must determine which target section $t'$ falls into. A common approach is to iterate through target sections or use binary search if sections are sorted, checking if the computed $t'$ from the formula falls within the current target section's bounds.*

3. **Compute the offset terms $a_1$ and $a_2$**:
   - Compute $a_2$ for the base segment:
     $$ a_2 = \sum_{j=1}^{n_l-1} \bar{\mu}_j (t_{\max, j} - t_{\min, j}) - \bar{\mu}_{n_l} t_{\min, n_l} $$
   - Compute $a_1$ for the target segment:
     $$ a_1 = \sum_{i=1}^{n_{l'}-1} \bar{\mu}'_i (t'_{\max, i} - t'_{\min, i}) - \bar{\mu}'_{n_{l'}} t'_{\min, n_{l'}} $$
   *Note: The sums are over all sections preceding the containing section. If $n_l=1$ or $n_{l'}=1$, the sum is empty (0).*

4. **Calculate the offset $R$**:
   $$ R = \frac{C \cdot a_2 - a_1}{\bar{\mu}'_{n_{l'}}} $$

5. **Compute the mapped depth $t'$**:
   $$ t' = C \cdot \frac{\bar{\mu}_{n_l}}{\bar{\mu}'_{n_{l'}}} \cdot t + R $$

### Formulas
The core relationship is derived from equating the majorant optical depths:
$$ \bar{\tau}'(t') = C \cdot \bar{\tau}(t) $$

Expanding the piecewise linear majorant optical depth functions:
$$ \sum_{i=1}^{M} \bar{\mu}'_i (\min(t', t'_{\max, i}) - t'_{\min, i}) = C \cdot \sum_{j=1}^{N} \bar{\mu}_j (\min(t, t_{\max, j}) - t_{\min, j}) $$

For a specific section $n_{l'}$ containing $t'$ and $n_l$ containing $t$, this simplifies to:
$$ \bar{\mu}'_{n_{l'}} \cdot t' + a_1 = C \cdot [\bar{\mu}_{n_l} \cdot t + a_2] $$

Solving for $t'$:
$$ t' = C \cdot \frac{\bar{\mu}_{n_l}}{\bar{\mu}'_{n_{l'}}} \cdot t + R $$

Where:
- $a_1 = \sum_{i=1}^{n_{l'}-1} \bar{\mu}'_i (t'_{\max, i} - t'_{\min, i}) - \bar{\mu}'_{n_{l'}} t'_{\min, n_{l'}}$
- $a_2 = \sum_{j=1}^{n_l-1} \bar{\mu}_j (t_{\max, j} - t_{\min, j}) - \bar{\mu}_{n_l} t_{\min, n_l}$
- $R = \frac{C \cdot a_2 - a_1}{\bar{\mu}'_{n_{l'}}}$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def majorant_optical_depth_shift(t, base_sections, target_sections, tau_max_base, tau_max_target):
    """
    Maps interaction depth t from base segment to target segment.
    
    Args:
        t: float, interaction depth on base segment.
        base_sections: list of dicts/objects with keys 't_min', 't_max', 'mu_bar'.
        target_sections: list of dicts/objects with keys 't_min', 't_max', 'mu_bar'.
        tau_max_base: float, total majorant optical depth of base segment.
        tau_max_target: float, total majorant optical depth of target segment.
        
    Returns:
        t_prime: float, mapped interaction depth on target segment.
    """
    if tau_max_base == 0:
        return 0.0  # Avoid division by zero; no scattering possible.
        
    C = tau_max_target / tau_max_base
    
    # Find base section index n_l containing t
    n_l = -1
    for j, section in enumerate(base_sections):
        if section['t_min'] <= t < section['t_max']:
            n_l = j
            break
    if n_l == -1:
        # Handle edge case: t is exactly at the last section's exit or outside
        # Assuming t is within the total length of the base segment
        n_l = len(base_sections) - 1
        if t >= base_sections[-1]['t_max']:
            t = base_sections[-1]['t_max'] - 1e-9 # Clamp to inside
            
    # Compute a2
    a2 = 0.0
    for j in range(n_l):
        section = base_sections[j]
        a2 += section['mu_bar'] * (section['t_max'] - section['t_min'])
    a2 -= base_sections[n_l]['mu_bar'] * base_sections[n_l]['t_min']
    
    # We need to find n_l' such that the computed t' falls within target_sections[n_l']
    # Since t' depends on n_l', we must iterate or search.
    # A simple approach: iterate through all target sections, compute t' assuming that section,
    # and check if t' is within that section's bounds.
    
    t_prime = None
    n_l_prime = -1
    
    for i, target_section in enumerate(target_sections):
        # Assume t' is in this section
        # Compute a1 for this section
        a1 = 0.0
        for k in range(i):
            sec = target_sections[k]
            a1 += sec['mu_bar'] * (sec['t_max'] - sec['t_min'])
        a1 -= target_section['mu_bar'] * target_section['t_min']
        
        # Compute R
        R = (C * a2 - a1) / target_section['mu_bar']
        
        # Compute t'
        t_candidate = C * (base_sections[n_l]['mu_bar'] / target_section['mu_bar']) * t + R
        
        # Check if t_candidate is within this target section
        if target_section['t_min'] <= t_candidate < target_section['t_max']:
            t_prime = t_candidate
            n_l_prime = i
            break
            
    if t_prime is None:
        # Fallback: if no section contains the computed t', clamp to the last section or handle error
        # This might happen due to numerical precision or if the ray exits the volume
        t_prime = target_sections[-1]['t_max'] - 1e-9
        
    return t_prime
```

### Edge Cases and Pitfalls
- **Zero Total Optical Depth**: If $\bar{\tau}_{\max} = 0$, the scaling constant $C$ is undefined. This implies no scattering occurs, and the shift is trivial or irrelevant.
- **Section Boundary Precision**: Floating-point errors may cause $t'$ to fall slightly outside the computed section bounds. Clamping or re-evaluating the section index may be necessary.
- **Ray Exit**: If the mapped depth $t'$ falls outside the target segment's volume bounds, the interaction is invalid for the target view.
- **Non-Matching Sections**: The number of sections $N$ and $M$ may differ. The algorithm handles this by iterating through target sections to find the correct one.

## Operating characteristics
- **Time Complexity**: $O(N + M)$ per shift, where $N$ and $M$ are the number of volume sections in the base and target segments, respectively. This is because we iterate through sections to find the containing ones and compute the sums $a_1$ and $a_2$.
- **Memory Complexity**: $O(1)$ additional memory if the section data is accessed sequentially, or $O(N + M)$ to store the section data.
- **Strengths**: Provides an exact analytical mapping for piecewise constant majorant extinction coefficients. Efficient for multi-view rendering where many paths are reused.
- **Limitations**: Requires pre-computation of section boundaries and majorant coefficients. Assumes piecewise constant extinction, which may not hold for highly heterogeneous media without sufficient subdivision.
- **When to Use**: When reusing paths between multiple views with different camera positions or different volume geometries/extinction coefficients.
- **When Not to Use**: For single-view rendering where no shift is needed, or for media where the majorant extinction coefficient varies continuously and cannot be well-approximated by piecewise constants.

## Background
The majorant optical depth shift is derived in the context of volumetric multi-view rendering, specifically for reusing light transport paths between different views. It was introduced in the paper "Volumetric Multi-View Rendering" [S1]. The method allows for efficient rendering by avoiding redundant sampling of the medium when the view changes, provided the medium's properties can be described by piecewise constant majorant extinction coefficients.

## Sources
[S1] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
