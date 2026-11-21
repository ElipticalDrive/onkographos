# Optimized Monte Carlo Integrator

Also called: Cached Monte Carlo Integration, Optimized Monte Carlo Integration, Variable Caching Integrator

## Overview
The Optimized Monte Carlo Integrator is a method for computing radiance scattered inside a homogeneous, box-shaped participating medium from a collimated light source towards the camera [S1]. It solves the problem of redundant geometric traversal in single-scattering estimation by caching segment properties and using importance sampling proportional to the geometry-dependent transmittance [S1]. It sets itself apart from standard ray marching by pre-computing a cumulative distribution histogram of trapezoidal segments to select interaction points directly, and from Delta Tracking by exploiting the specific convex geometry of a box to allow analytic integration of unoccluded paths [S1].

## How it works
**Inputs:**
*   Camera ray origin $p_o$ and direction $\omega_o$.
*   Light direction $\omega_i$ (collimated).
*   Medium extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, and phase function $f(\omega_i, \omega_o)$.
*   Box geometry defined by entry and exit points for light and camera rays.
*   Number of samples $N_{sample}$.

**Outputs:**
*   Estimated radiance $L$.

**Data Structures:**
*   **Segment Cache Matrix $K$**: A matrix where each row $s$ corresponds to a trapezoidal segment formed by connecting light and camera ray intersections with the box boundaries. Fields:
    *   $K_{s,1}$: Upper bound integral part, $1 - E_u$.
    *   $K_{s,2}$: Slope, $\sigma_t c$ (where $c$ is a constant related to the geometry).
    *   $K_{s,3}$: Size of segment along camera ray, $t_{C,B} - t_{C,S}$.
    *   $K_{s,4}$: Starting position along camera ray, $t_{C,S}$.
*   **Cumulative Histogram $H$**: Array of size $N_{seg}$ (number of segments).
*   **Multiplicative Weights $D$**: Array of size $N_{seg}$.
*   **Summands $M$**: Array of size $N_{seg}$.

**Steps:**

1.  **Geometry Traversal and Caching:**
    Traverse the edges of the box to identify trapezoidal segments connecting the incident light exit points and outgoing camera exit points. For each segment $s$:
    *   Compute the transmittance $T_s$ for the segment.
    *   Compute total transmittance $T_{total} = \sum T_s$.
    *   Cache segment parameters into $K_s$:
        $$ K_{s,1} = 1 - E_u $$
        $$ K_{s,2} = \sigma_t c $$
        $$ K_{s,3} = t_{C,B} - t_{C,S} $$
        $$ K_{s,4} = t_{C,S} $$
    [S1]

2.  **Build Cumulative Distribution:**
    *   Compute selection probability for segment $k$:
        $$ G_k = \frac{T_k}{T_{total}} $$
    [S1]
    *   Compute multiplicative weight:
        $$ D_k = \frac{1}{G_k} $$
    [S1]
    *   Compute summand:
        $$ M_k = -H_{k-1} \cdot D_k $$
        (Initialize $H_0 = 0$).
    [S1]
    *   Build cumulative histogram:
        $$ H_k \leftarrow \sum_{i=1}^k G_i $$
    [S1]

3.  **Sampling Loop (for $n = 1$ to $N_{sample}$):**
    *   Draw uniform random value:
        $$ r \leftarrow \xi\left(\frac{k}{N_{sample}}\right) $$
        where $\xi$ is a uniform random number in $[0, 1]$.
    [S1]
    *   Select segment $s$ by comparing $r$ against the histogram step functions:
        $$ s \leftarrow \text{step}(H_1, r) + \text{step}(H_2, r) $$
        (This implies finding the index $s$ such that $H_{s-1} \le r < H_s$).
    [S1]
    *   Transform sample into segment coordinates:
        $$ r_s = r \cdot D_s + M_s $$
    [S1]
    *   Sample distance $t$ along the camera ray within segment $s$ using importance sampling derived from the analytic integration (see Sec. 5.1 of [S1]). The specific distance sampling formula depends on the analytic transmittance distribution of the trapezoid, but the key is that the exponential term is cancelled by the PDF.
    [S1]
    *   Accumulate visibility $V_{total}$:
        $$ V_{total} = V_{total} + V(p_o - t \omega_o) $$
        where $V$ is the visibility function (1 if unoccluded, 0 otherwise). The exponential transmittance is implicitly handled by the importance sampling PDF.
    [S1]

4.  **Final Radiance Computation:**
    $$ L = \frac{1}{N_{sample}} \sigma_s f(\omega_i, \omega_o) V_{total} T_{total} L_i $$
    where $L_i$ is the incident light radiance.
    [S1]

**Edge Cases and Pitfalls:**
*   **Single Scattering Assumption:** The analytic integration and the specific ratio estimator formulation are exact only for single scattering. For multiple scattering, this method serves as a base or requires approximation [S1].
*   **Box Geometry:** The method relies on the medium being a convex box. Non-convex or arbitrary geometries require different traversal or delta tracking [S1].
*   **Precision:** The calculation of $M_k$ involves subtraction of potentially large numbers if $H_{k-1}$ is large and $D_k$ is large; care must be taken with floating-point precision in the histogram construction.

## Subtopics

### [Backward Monte Carlo](15115-backward-monte-carlo.md)
Backward Monte Carlo (BMC) traces photons from the camera to the light source, offering stability for in-situ rendering inside the medium compared to Forward Monte Carlo (FMC). It avoids the $|V_{s \to c}|^{-2}$ variance explosion inherent in FMC when the camera is inside the scattering medium because the light source (e.g., sun) is effectively at infinity. While the Optimized Monte Carlo Integrator described above is a forward-looking optimization for box geometries, BMC is a general strategy for handling the geometry term variance in participating media.

### Rushmeier's Zonal-Monte Carlo Method
A hybrid approach where a zonal method is used in a first pass to compute source radiances, followed by a Monte Carlo second pass in the rendering step to handle anisotropy for the single scattering approximation, accounting for one extra directional bounce [S2]. This differs from the pure Optimized Monte Carlo Integrator by separating the radiance computation from the directional scattering approximation.

## Operating characteristics
*   **Time Complexity:** $O(N_{seg} + N_{sample})$. The geometry traversal is $O(N_{seg})$ (constant for a box, as the number of segments is fixed by the box topology relative to the ray directions), and sampling is $O(N_{sample})$ with $O(1)$ segment selection via binary search or linear scan of the small histogram.
*   **Memory Complexity:** $O(N_{seg})$ to store the cache matrix $K$, histogram $H$, weights $D$, and summands $M$. For a box, $N_{seg}$ is small and constant.
*   **Performance:** Delivers significant performance improvements over ray marching for real-time applications, particularly in mixed reality and AR setups where additive lighting effects are needed [S1].
*   **Strengths:** Eliminates redundant geometric calculations per sample; exact analytic integration for unoccluded paths reduces variance; suitable for real-time rendering.
*   **Limitations:** Restricted to homogeneous, box-shaped media; primarily designed for single scattering; collimated light sources only.
*   **When to use:** Real-time rendering of light shafts in homogeneous fog/clouds within a defined volume, especially in AR/VR where performance is critical.
*   **When not to use:** Heterogeneous media, multiple scattering, non-box geometries, or point lights.

## Background
The method was introduced in "Collimated Whole Volume Light Scattering in Homogeneous Finite Media" [S1]. It was developed to address the computational cost of ray marching in real-time rendering applications, specifically targeting mixed reality and optical see-through AR headsets [S1]. The work combines analytic integration of unoccluded single scattering, importance sampling matching the geometry-dependent transmittance, and variable caching to optimize the Monte Carlo integration process [S1].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
