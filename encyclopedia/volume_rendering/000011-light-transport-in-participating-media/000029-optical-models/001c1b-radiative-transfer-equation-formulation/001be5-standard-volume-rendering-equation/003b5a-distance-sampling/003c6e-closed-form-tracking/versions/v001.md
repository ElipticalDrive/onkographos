# Closed-form Tracking

Also called: Analytical Free-Path Sampling, Inverse Transform Sampling for Free Paths

## Overview
Closed-form Tracking is a Monte Carlo technique for sampling the free-flight distance in participating media where the extinction coefficient $\mu_t$ follows a simple analytical form (constant, polynomial, or exponential) [S1]. It solves the problem of determining the distance to the next collision by applying inverse transform sampling directly to the transmittance function [S1]. It is distinguished from [Whole Volume Distance Sampling](9002-whole-volume-distance-sampling.md) by its reliance on local analytical invertibility of the optical depth integral rather than global volume statistics, allowing for exact sampling without rejection steps when the cumulative distribution function (CDF) is analytically invertible [S1].

## How it works
**Inputs:**
- Ray parameterization $x(t) = o + t \cdot d$, where $o$ is the origin, $d$ is the direction, and $t \ge 0$ is the distance.
- Extinction coefficient function $\mu_t(x)$, which must be constant, polynomial, or exponentially varying [S1].
- A uniform random number $\zeta \in [0, 1]$ [S1].

**Outputs:**
- Sampled free-path distance $t_0$ [S1].

**Data Structures:**
- No complex data structures are required beyond the ray parameters and the coefficients defining $\mu_t(x)$ [S1].

**Steps:**
1. Define the transmittance function $T(t)$ along the ray:
   $$ T(t) = \exp\left(-\int_0^t \mu_t(x(\tau)) d\tau\right) $$
   The probability density function (PDF) for the free path is derived from this transmittance [S1].

2. Determine the Cumulative Distribution Function (CDF), $F(t)$, associated with the PDF. The CDF represents the probability that a collision occurs before distance $t$ [S1].

3. Sample the distance $t_0$ based on the invertibility of $F(t)$:
   - **Case A: Analytically Invertible CDF.** If $F(t)$ can be inverted analytically, compute $t_0$ directly using the inverse CDF $F^{-1}$:
     $$ t_0 = F^{-1}(1 - \zeta) $$
     For a homogeneous volume with constant extinction $\mu_t$, the optical thickness is $\tau(t) = \mu_t t$. The transmittance is $T(t) = e^{-\mu_t t}$. The CDF is $F(t) = 1 - e^{-\mu_t t}$. Inverting this yields the closed-form expression [S1]:
     $$ t_0 = -\frac{\log(1 - \zeta)}{\mu_t} $$
     Since $\zeta$ is uniform in $[0, 1]$, $1-\zeta$ is also uniform, so this is often implemented as $t_0 = -\frac{\log(\zeta)}{\mu_t}$ [S1].

   - **Case B: Non-Invertible CDF but Analytic Optical Thickness.** If $F(t)$ is not analytically invertible, but the optical thickness integral $\tau(t) = \int_0^t \mu_t(x(\tau)) d\tau$ has an analytic solution, solve for $t_0$ iteratively using Newton's method [S1]. The goal is to find $t$ such that:
     $$ 1 - \exp(-\tau(t)) = 1 - \zeta \implies \tau(t) = -\log(\zeta) $$
     Define $g(t) = \tau(t) + \log(\zeta)$. Use Newton's iteration:
     $$ t_{n+1} = t_n - \frac{g(t_n)}{g'(t_n)} = t_n - \frac{\tau(t_n) + \log(\zeta)}{\mu_t(x(t_n))} $$
     Iterate until convergence [S1].

**Parameters:**
- $\mu_t$: Extinction coefficient. For homogeneous media, this is a scalar constant [S1]. For heterogeneous media, it is a function of position [S1].
- $\zeta$: Uniform random number in $[0, 1]$ [S1].

**Edge Cases and Pitfalls:**
- **Precision:** When $\zeta$ is very close to 0 or 1, logarithmic operations may suffer from floating-point precision issues. Using $1-\zeta$ or clamping $\zeta$ to $[\epsilon, 1-\epsilon]$ is recommended [G].
- **Non-invertibility:** If neither the CDF nor the optical thickness integral is analytically solvable, this method cannot be applied directly, and numerical root-finding or alternative sampling methods (like [Whole Volume Distance Sampling](9002-whole-volume-distance-sampling.md)) are required [S1].

## Subtopics

### Regular Tracking
Regular Tracking is a variant of closed-form tracking applied to piecewise homogeneous volumes [S1]. It handles media composed of distinct regions with constant extinction coefficients by visiting boundaries along the ray [S1]. The algorithm integrates the optical thickness of individual regions sequentially. It stops when the negative exponentiated value of the integrated optical thickness reaches a stochastically chosen value of transmittance (i.e., when the accumulated transmittance matches $1-\zeta$) [S1]. This method is known as surface tracking in some contexts [S1]. It becomes expensive if boundaries are not discovered quickly, such as in unstructured grids, but is efficient with hierarchical structures like grids or octrees [S1].

## Operating characteristics
- **Time Complexity:** $O(1)$ for homogeneous or simple analytical volumes where the inverse CDF is closed-form [S1]. $O(k)$ for Regular Tracking, where $k$ is the number of boundaries crossed, assuming $O(1)$ boundary intersection tests [S1].
- **Memory Complexity:** $O(1)$, as it requires only local coefficient storage [S1].
- **Strengths:** Unbiased, exact sampling without rejection; computationally efficient for simple media; no need for global volume statistics [S1].
- **Limitations:** Limited to media with analytically integrable extinction functions; Regular Tracking variant is slow if boundary traversal is expensive [S1].
- **When to use:** Ideal for homogeneous media, exponential media, or piecewise homogeneous media with efficient boundary traversal [S1].
- **When not to use:** For complex heterogeneous media where $\mu_t(x)$ does not allow analytic integration or inversion [S1].

## Background
Closed-form tracking relies on inverse transform sampling, a technique first applied to Monte Carlo integration by Ulam [S1]. The specific application to volume rendering for simple extinction profiles is detailed in spectral and decomposition tracking literature [S1]. Regular Tracking (or surface tracking) was introduced by Sutton et al. [S1] and further discussed by Leppänen [S1]. Newton's method for solving the optical thickness equation was referenced from Brown and Martin [S1].

## Sources
[S1] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
