# Parameter Range Culling

Also called: PIC culling, Run Time Parameter Range Culling

## Overview
Parameter Range Culling (PRC) is a runtime visibility optimization that isolates particle subsets by culling particles whose scalar parameter values (e.g., density, velocity) do not overlap a user-defined valid range [S1]. It sets itself apart from static culling by performing these checks dynamically during traversal, allowing for interactive exploration of simulation data without pre-processing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A macrocell hierarchy of particle packets, where each macrocell stores the minimum and maximum values of a specific scalar parameter for all particles it contains [S1]. A user-defined valid range $[R_{min}, R_{max}]$ for the parameter [S1].
*   **Output:** A subset of particles that satisfy the range constraint, reducing the number of ray/sphere intersection tests performed [S1].

**Data Structures:**
*   **Macrocell:** Contains bounding geometry for the packet and two scalar values: $P_{min}$ and $P_{max}$, representing the minimum and maximum parameter values of all particles within that macrocell [S1].

**Steps:**
1.  **Macrocell-Level Culling:** During traversal of the macrocell hierarchy, compare the macrocell's stored range $[P_{min}, P_{max}]$ against the user-defined valid range $[R_{min}, R_{max}]$ [S1].
    *   If the intervals do not overlap (i.e., $P_{max} < R_{min}$ or $P_{min} > R_{max}$), the macrocell is skipped entirely [S1].
    *   This check is amortized over the number of particles in the macrocell, making it significantly cheaper than individual intersection tests [S1].
2.  **Particle-Level Culling:** If the macrocell's range overlaps with the valid range, traverse the individual particles within the macrocell [S1].
    *   For each particle, check if its specific parameter value $p$ lies within $[R_{min}, R_{max}]$ [S1].
    *   Only perform the ray/sphere intersection test if $R_{min} \le p \le R_{max}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parameter_range_culling(macrocell, ray, r_min, r_max):
    # Step 1: Macrocell-level culling
    # Check if the macrocell's parameter range [P_min, P_max] overlaps with [r_min, r_max]
    if macrocell.p_max < r_min or macrocell.p_min > r_max:
        return [] # No particles in this macrocell can satisfy the condition
    
    # Step 2: Particle-level culling
    valid_particles = []
    for particle in macrocell.particles:
        # Check if the individual particle's parameter value is within the valid range
        if r_min <= particle.parameter_value <= r_max:
            # Perform intersection test only for valid particles
            if ray.intersects_sphere(particle.position, particle.radius):
                valid_particles.append(particle)
    return valid_particles
```

**Parameters:**
*   $R_{min}, R_{max}$: User-defined bounds for the scalar parameter [S1].
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **SIMD Overhead:** Implementing range checks efficiently using SIMD instructions adds computational overhead. Specifically, 16 SIMD operations are required to implement the macrocell and particle range checking [S1].
*   **Performance Degradation:** This overhead decreases performance by a factor of 1.38–2.28 compared to preprocessed versions of the data where culling is static [S1].
*   **Grid Bounds:** Some performance difference is attributed to grid bounds; smaller bounds lead to fewer packet traversals, which can interact with the culling efficiency [S1].

## Operating characteristics
*   **Time Complexity:** The macrocell check is $O(1)$ per macrocell, amortized over the particles within it [S1]. The particle check is $O(N)$ for $N$ particles in the macrocell, but reduces the constant factor by skipping intersection tests for out-of-range particles [S1].
*   **Memory Complexity:** Requires storing $P_{min}$ and $P_{max}$ per macrocell [S1].
*   **Measured Performance:**
    *   **Thunder Dataset:** Reduces total processing time by 43.00%. Mean culling rate is 14.23%, max culling rate is 43.06% [S1].
    *   **BulletTorso Dataset:** Reduces total processing time by 34.17%. Mean culling rate is 3.17%, max culling rate is 11.18% [S1] (Table VIII in [S1]).
*   **Strengths:** Provides flexibility for data analysis by allowing dynamic isolation of particle subsets without pre-processing [S1].
*   **Limitations:** The runtime overhead of SIMD range checks can degrade interactive performance compared to static preprocessed data [S1].
*   **When to use:** When interactive exploration of simulation data based on scalar parameters (e.g., temperature, velocity) is required [S1].
*   **When not to use:** When maximum performance is critical and the parameter ranges are static, as preprocessed culling is faster [S1].

## Background
*   **Origin:** Introduced in "A Coherent Grid Traversal Approach to Visualizing Particle Data" by Thiago Ize et al. [S1].
*   **History:** Developed as part of the Particle Coherent Grid Traversal (PCGT) framework to support interactive visualization of large-scale particle simulations [S1].
*   **Usage:** Used in scientific visualization for analyzing particle-based simulations, such as fluid dynamics or astrophysical data, where researchers need to filter particles based on physical properties [S1].

## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
