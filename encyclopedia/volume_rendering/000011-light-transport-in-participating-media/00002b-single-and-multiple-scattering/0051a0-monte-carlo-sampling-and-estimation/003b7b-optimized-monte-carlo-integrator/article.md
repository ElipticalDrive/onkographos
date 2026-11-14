# Optimized Monte Carlo Integrator

Also called: Cached Monte Carlo Integration, Optimized Monte Carlo Integration, Variable Caching Integrator

## Overview
The Optimized Monte Carlo Integrator is a method for computing radiance scattered inside a homogeneous participating medium from a collimated light source towards the camera, specifically targeting real-time rendering applications [S1]. It solves the problem of redundant geometric traversal by caching segment data and using importance sampling proportional to geometry-dependent transmittance [S1]. It sets itself apart from standard ray marching by avoiding repeated box outline traversal and from Delta Tracking by relying on analytic integration of unoccluded single scattering within a bounded box geometry [S1].

## How it works
The method computes single-scattering radiance in a box-shaped medium. It assumes a collimated light source (e.g., distant star or gobo-masked light) and a camera ray. The core innovation is the decomposition of the medium's geometry into trapezoidal segments and the caching of integration variables to allow efficient importance sampling [S1].

### Inputs and Outputs
*   **Inputs**: Camera ray origin $p_o$ and direction $\omega_o$; Light direction $\omega_i$; Medium extinction coefficient $\sigma_t$; Medium scattering coefficient $\sigma_s$; Phase function $f(\omega_i, \omega_o)$; Light intensity $L_i$; Number of samples $N_{sample}$; Box geometry defined by entry and exit points for light and camera rays.
*   **Outputs**: Estimated radiance $L$ at the camera pixel.

### Data Structures
The method relies on a cached matrix $K$ storing precomputed values for each trapezoidal segment $k$. The matrix $K$ has dimensions $N_{segments} \times 4$.
*   $K_{k,1}$: Upper bound integral part, value $1 - E_u$.
*   $K_{k,2}$: Slope, value $\sigma_t c$ (where $c$ is a geometric constant related to the segment).
*   $K_{k,3}$: Size of segment along camera ray, value $t_{C,B} - t_{C,S}$.
*   $K_{k,4}$: Starting position along the camera ray, value $t_{C,S}$.

Additionally, a cumulative distribution histogram $H$ and a multiplicative weight array $D$ are stored for segment selection.

### Steps

1.  **Geometry Traversal and Segment Decomposition**:
    Traverse the edges of the box from corner to corner to connect the incident light exit points and outgoing camera exit points. This decomposes the medium into trapezoid-shaped segments [S1].

2.  **Cache Unoccluded Integration Values**:
    For each segment $k$, compute and cache the values in matrix $K$ as defined above. This involves calculating the analytic integration of unoccluded single scattering for the segment [S1].

3.  **Build Cumulative Histogram**:
    Compute the probability of selecting segment $k$, denoted $G_k$, as the ratio of the segment's transmittance contribution $T_k$ to the total transmittance $T_{total}$:
    $$ G_k = \frac{T_k}{T_{total}} $$
    Sum probabilities to build the cumulative histogram $H$:
    $$ H_k \leftarrow \sum_{i=1}^k G_i $$
    Compute the multiplicative weight $D_k$ as the inverse probability:
    $$ D_k = \frac{1}{G_k} $$
    Compute the summand $M_k$ using the previous CDF histogram value:
    $$ M_k \leftarrow -H_{k-1} \cdot D_k $$
    (Note: $H_0 = 0$) [S1].

4.  **Segment Selection**:
    For each sample $j$ from $1$ to $N_{sample}$:
    a. Draw a random value $r$ from a uniform distribution:
       $$ r \leftarrow \xi\left(\frac{k}{N_{sample}}\right) $$
       where $\xi$ is a uniform random number in $[0, 1]$ [S1].
    b. Find the selected segment $s$ by comparing $r$ against the step functions of the histogram $H$:
       $$ s \leftarrow \text{step}(H_1, r) + \text{step}(H_2, r) $$
       (This implies a binary search or linear scan through the histogram steps to find the interval containing $r$) [S1].

5.  **Transform Sample in Segment Coordinates**:
    Map the random value $r$ to the local coordinate system of the selected segment $s$:
    $$ r_s \leftarrow r \cdot D_s + M_s $$
    This $r_s$ represents the normalized position within the segment's transmittance distribution [S1].

6.  **Sample Distance**:
    Sample the distance $t$ along the camera ray according to the importance sampling distribution derived from the analytic integration (as detailed in the source's Sec. 5.1, which matches the geometry-dependent transmittance) [S1].

7.  **Accumulate Visibility**:
    In the integration loop, accumulate only the visibility term $V$. The exponential transmittance function is cancelled by the probability density function (PDF) of the importance sampling.
    $$ V_{total} \leftarrow V_{total} + V(p_o - t \omega_o) $$
    where $V(p_o - t \omega_o)$ is the visibility of the point at distance $t$ from the camera origin [S1].

8.  **Compute Final Radiance**:
    Re-weight the total visibility to compute the final radiance:
    $$ L = \frac{1}{N_{sample}} \sigma_s f(\omega_i, \omega_o) V_{total} T_{total} L_i $$
    where $T_{total}$ is the total transmittance factor [S1].

### Parameters
*   $N_{sample}$: Number of Monte Carlo samples. Typical values depend on the desired noise level and real-time constraints [S1].
*   Target Application: Real-time rendering [S1].

### Edge Cases and Pitfalls
*   **Single Scattering Limitation**: The method is derived for single scattering. The source notes that only in the single-scattering case does a ratio estimator with the same importance sampling scheme yield the same integration algorithm [S1].
*   **Box Geometry**: The method assumes a box-shaped medium. The decomposition into trapezoids relies on connecting box corners. If the medium is not box-shaped, this specific decomposition does not apply [S1].
*   **Collimated Light**: The method is optimized for collimated light sources. Non-collimated lights may require different handling [S1].
*   **Precision**: The cancellation of the exponential function by the PDF requires careful implementation to avoid numerical instability, though the source does not specify exact precision requirements [S1].

## Subtopics

### Rushmeier's Zonal-Monte Carlo Method
A variant where the zonal method is used in a first pass to compute source radiances, followed by a Monte Carlo second pass in the rendering step to handle anisotropy for the single scattering approximation, accounting for one extra directional bounce [S2].

## Operating characteristics
*   **Performance**: Delivers performance improvements over ray marching by avoiding repeated traversal of the box outline [S1].
*   **Complexity**: The complexity is dominated by the geometry traversal (once) and the Monte Carlo sampling ($N_{sample}$ steps). The caching reduces per-sample geometric cost to constant time after histogram construction [S1].
*   **Memory**: Requires storage for the cached matrix $K$ and histograms $H, D$, proportional to the number of trapezoidal segments [S1].
*   **Strengths**: Efficient for real-time rendering of box-shaped media with collimated light; reduces redundant computations [S1].
*   **Limitations**: Limited to single scattering; requires box-shaped geometry; assumes collimated light [S1].
*   **When to use**: Real-time applications involving box-shaped participating media and collimated light sources (e.g., light shafts in AR/VR) [S1].
*   **When not to use**: Multiple scattering scenarios; non-box geometries; non-collimated lights [S1].

## Background
The method was introduced in the paper "Collimated Whole Volume Light Scattering in Homogeneous Finite Media" [S1]. It is designed for mixed reality and real-time rendering applications, such as optical see-through AR headsets, where virtual light shafts are rendered as additive lighting effects [S1]. The technique builds on analytic integration of unoccluded single scattering and importance sampling strategies [S1]. Rushmeier's Zonal-Monte Carlo Method is an earlier related approach mentioned in surveys of participating media rendering [S2].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
