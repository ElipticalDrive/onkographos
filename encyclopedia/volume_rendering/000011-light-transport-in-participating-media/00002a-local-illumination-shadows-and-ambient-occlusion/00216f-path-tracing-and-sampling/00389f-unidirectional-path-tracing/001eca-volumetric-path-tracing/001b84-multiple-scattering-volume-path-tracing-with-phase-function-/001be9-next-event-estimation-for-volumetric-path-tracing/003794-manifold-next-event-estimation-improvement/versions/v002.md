# Manifold Next Event Estimation Improvement

Also called: Fermat's Principle NEE, Koerner et al. MNEE improvement, Volume-based Next Event Estimation Improvement

## Overview
Manifold Next Event Estimation (MNEE) Improvement is an optimization for volumetric path tracing that estimates direct lighting by searching for a point within the participating medium that satisfies Fermat’s principle, rather than walking along surface boundaries [S1]. It addresses the limitations of standard NEE and original MNEE in scenes with refractive boundaries and complex geometry, specifically targeting highly directional, near-delta light distributions [S1]. It builds on [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md) by refining the connection strategy for media surrounded by refractive boundaries [S1].

## How it works
The method improves upon Hanika et al.'s MNEE [S1]. Instead of walking along boundaries using geometric derivatives, it searches for a point in the volume that satisfies Fermat’s principle to connect the current path vertex to the light source [S1]. This avoids failures on detailed or displaced surfaces where derivative-based walking fails [S1]. The method can be integrated into a unidirectional path tracer using Multiple Importance Sampling (MIS) [S1].

**Inputs and Outputs:**
- Input: Current path vertex position, direction, participating medium properties, and light source geometry [S1].
- Output: A valid connection point on the light source and the transmittance along the path [S1].

**Data Structures:**
- Not given in the sources.

**Steps:**
1. **Fermat’s Principle Search:** Search for a point $\mathbf{x}_v$ within the volume such that the optical path length to the light source is stationary, satisfying Fermat’s principle [S1].
2. **Connection:** Connect the found volume point $\mathbf{x}_v$ to the light source [S1].
3. **Estimation:** Estimate the direct illumination contribution using the connection, accounting for transmittance through the medium and boundaries [S1].

**Formulas:**
- Fermat’s principle implies the variation of the optical path length $\delta \int n(\mathbf{r}) ds = 0$ [S1]. The exact optimization function is not given in the sources [S1].

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- The method is computationally expensive due to the optimization search [S1].
- It is suitable for highly directional, near-delta distributions but may be inefficient for other cases [S1].
- Original MNEE fails on detailed or displaced surfaces; the improvement mitigates this by searching in the volume, but the cost remains high [S1].

## Subtopics

### Manifold Exploration
Manifold exploration is a mutation strategy for Metropolis light transport that improves sampling for paths involving specular and highly glossy surfaces, and is usable for participating media surrounded by refractive boundaries [S1]. The manifold exploration idea further benefits next event estimation (NEE) by enabling connections through multiple refracting surfaces [S1].

## Operating characteristics
- **Time Complexity:** High computational overhead due to the search for a volume point satisfying Fermat’s principle [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Handles refractive boundaries better than standard NEE; avoids failures on detailed or displaced surfaces where derivative-based surface walking fails [S1].
- **Limitations:** Too expensive for general use; best suited for highly directional, near-delta light distributions [S1].
- **When to use:** Scenes with participating media surrounded by refractive boundaries where standard NEE fails and surface-based MNEE is unstable or too costly due to geometry complexity [S1].
- **When not to use:** Scenes with low-order scattering or non-directional lighting where the overhead is unjustified [S1].

## Background
Standard NEE is unsuitable for participating media with refractive boundaries [S1]. Hanika et al. proposed manifold next event estimation (MNEE), which leverages manifold exploration to connect to the light source through multiple refracting surfaces [S1]. The main drawback of MNEE is that the search for the boundary vertex may not succeed, and it relies on geometric derivatives that do not handle detailed or displaced surfaces well [S1]. Koerner et al. improved MNEE by searching for a point in the volume which satisfies Fermat’s principle instead of walking over the surface [S1]. Both methods can be integrated into a unidirectional path tracer using multiple importance sampling (MIS) [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
