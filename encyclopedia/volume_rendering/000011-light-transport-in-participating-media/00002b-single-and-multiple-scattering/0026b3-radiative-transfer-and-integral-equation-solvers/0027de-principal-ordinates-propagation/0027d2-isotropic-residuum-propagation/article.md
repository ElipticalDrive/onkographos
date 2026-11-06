# Isotropic Residuum Propagation

Also called: IR propagation

## Overview
Isotropic Residuum Propagation (IR propagation) is a secondary propagation stage within Principal-Ordinates Propagation that rapidly transports remaining isotropic energy after the initial anisotropic phase. It solves the problem of high computational cost in optically thick media by switching from expensive directional propagation to a cheaper single-grid isotropic scheme once radiance anisotropy falls below a threshold. It sets itself apart from full anisotropic propagation by merging multiple directional grids into one global grid, achieving significant speed-ups with negligible visual difference for the residuum energy.

## How it works
**Inputs and Outputs:**
- **Inputs:** Radiance fields from the anisotropic propagation stage, phase function anisotropy $g$, scattering coefficient $\sigma_s$, grid spacing $\Delta x$, anisotropy threshold $\varepsilon_a$, and initial radiance anisotropy $a_b$.
- **Outputs:** Final radiance field in a single global grid.

**Data Structures:**
- **Global Grid:** A single 3D grid aligned with the medium, typically at double the resolution of individual directional propagation grids to ensure sufficient penetration depth from the first row of cells [S1].
- **Radiance Field:** Stored as scalar values per cell, as directionality is no longer tracked.

**Parameters:**
- $\varepsilon_a$: Small anisotropy threshold. Typical value: around 0.1 or higher [S1].
- $\Delta x$: Grid spacing. Example: 0.25 m [S1].
- $g$: Phase function anisotropy.
- $\sigma_s$: Scattering coefficient.
- $a_b$: Initial maximum radiance anisotropy (e.g., 1 for directional light, lower if prefiltered) [S1].

**Algorithm Steps:**
1. **Initialization:** Determine initial parameters $a_b$ and $g$ at initialization. $a_b$ may be reduced by prefiltering incident radiance [S1].
2. **Calculate Switch Iteration ($m_0$):** Compute the number of anisotropic iterations after which to switch to isotropic propagation using the Henyey-Greenstein (HG) self-convolution property [S1]:
   $$ m_0 = \frac{\ln \varepsilon_a - \ln a_b}{\ln g} \cdot \frac{1}{\sigma_s \cdot \Delta x} $$
   [S1]
3. **Anisotropic Propagation:** Perform full anisotropic propagation for $m_0$ iterations using multiple directional grids aligned with principal ordinates [S1].
4. **Check Anisotropy:** After $m_0$ iterations, verify if all propagation grids satisfy $|a_{n,i}| < \varepsilon_a$ for all cells $i$ and ordinates $n$ [S1]. If not, continue anisotropic propagation or adjust $m_0$ [S1].
5. **Merge Grids:** Combine energy from all directional grids into a single global grid aligned with the medium. This is valid because directionality is negligible [S1].
6. **Isotropic Residuum Propagation:** Propagate the merged residuum energy in the single global grid for additional iterations until the residual energy falls below a threshold $\varepsilon_L$ or convergence is reached [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def isotropic_residuum_propagation(radiance_fields, g, sigma_s, delta_x, epsilon_a, epsilon_L):
    # Step 1: Determine initial anisotropy
    # a_b is the maximum radiance anisotropy at initialization
    # For directional light, a_b = 1; prefiltering may reduce it
    a_b = get_initial_radiance_anisotropy(radiance_fields)
    
    # Step 2: Calculate switch iteration m0
    # Using HG self-convolution property
    # Note: ln(g) is negative for g < 1, so the formula yields positive m0
    numerator = math.log(epsilon_a) - math.log(a_b)
    denominator = math.log(g) * sigma_s * delta_x
    m0 = numerator / denominator
    
    # Step 3: Anisotropic Propagation (handled by parent method)
    # Perform m0 iterations of anisotropic propagation
    radiance_fields = anisotropic_propagation(radiance_fields, m0)
    
    # Step 4: Check anisotropy threshold
    # Verify if |a_{n,i}| < epsilon_a for all cells and ordinates
    max_anisotropy = get_max_anisotropy(radiance_fields)
    if max_anisotropy >= epsilon_a:
        # Continue anisotropic propagation or adjust m0
        # [G] In practice, m0 is often a heuristic; if threshold not met, 
        # more anisotropic iterations may be needed.
        pass
    
    # Step 5: Merge grids into single global grid
    # Use double resolution of individual grids for global grid
    global_grid = merge_directional_grids(radiance_fields, resolution_factor=2)
    
    # Step 6: Isotropic Residuum Propagation
    # Propagate in single grid until residual energy < epsilon_L
    while get_residual_energy(global_grid) > epsilon_L:
        global_grid = isotropic_transport_step(global_grid, sigma_s, delta_x)
        
    return global_grid
```

**Edge Cases and Pitfalls:**
- **Precision:** Logarithms in $m_0$ calculation require $a_b > 0$ and $g < 1$. If $a_b$ is very small, $m_0$ may be negative or zero, requiring a minimum iteration count [S1].
- **Boundaries:** Global grid resolution must be sufficient to allow light penetration from the first row of cells [S1].
- **Degenerate Input:** If $g=1$ (pure forward scattering), $\ln g = 0$, causing division by zero. This case is not handled in the formula [S1].

## Subtopics

### Multiple Propagation Grids for Environment Lighting
This variant handles distant environment illumination by using multiple superimposed, relatively small propagation grids oriented along different principal directions [S1]. It addresses the issue of high computational cost for environment maps by prefiltering incident radiance for each grid's finite solid angle to avoid singularity artifacts during initialization [S1]. The grids are combined for rendering, and the isotropic residuum propagation stage remains applicable [S1].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** IR propagation significantly reduces computation time. In the dragon dataset (milk, $\sigma_s = \{0.91, 1.07, 1.25\} m^{-1}$, $g = 0.95$, albedo > 99.9%), full propagation (64 ordinates, $16^3$ grid) took 120 ms and 90 ms for ~100 iterations. IR propagation ($m_0=20$, $32^3$ grid) took 28 ms and 24 ms, a ~4-fold speed-up [S1]. Typical setting for residuum propagation takes 2 ms [S1].
- **Memory:** Requires a single global grid for the residuum stage, reducing memory overhead compared to maintaining multiple directional grids [S1].

**Measured Performance:**
- Hardware: Not specified in sources [S1].
- Resolution: Dragon dataset used $16^3$ for anisotropic and $32^3$ for global grid [S1].
- Scene: Milk medium with high albedo [S1].

**Scaling:**
- Media with higher optical thickness decrease anisotropy faster, allowing earlier switch to isotropic mode [S1].
- Isotropic transport allows lower-resolution propagation domains due to low-frequency lighting frequencies [S1].

**Strengths and Limitations:**
- **Strengths:** Efficient handling of optically thick anisotropic media; significant speed-up with negligible visual difference [S1].
- **Limitations:** Heuristic nature of $m_0$ calculation; potential bias from empirical heuristics [S1].

**When to Use:**
- Use for optically thick media where anisotropy decays rapidly.
- Use when real-time performance is critical and slight visual approximation is acceptable [S1].

**When Not to Use:**
- Avoid for scenes requiring precise preservation of high-anisotropy effects in deep scattering regions [S1].

## Background
**Origin:**
- Proposed by Elek et al. in 2014 as part of Principal-Ordinates Propagation [S1].

**History and Development:**
- The method builds on the observation that scattering reduces radiance anisotropy, allowing a switch to isotropic propagation [S1]. It is similar to merging accumulation grids but applied to radiance fields [S1].

**Usage:**
- Used in real-time rendering of participating media, including direct illumination, low-order scattering, and arbitrary multiple scattering [S1].

## Sources
[S1] Principal-Ordinates Propagation for Real-Time Rendering ..., https://cg.ivd.kit.edu/publications/2015/principal-ordinates/ElekCAG2014.pdf, fetched 2026-10-29
