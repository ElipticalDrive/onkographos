# Free-path Sampling Methods

Also called: Free-flight sampling, flight distance sampling

## Overview
Free-path sampling methods are Monte Carlo techniques used to determine the distance to the next interaction (collision or absorption) along a ray in participating media. They solve the problem of sampling the random variable representing the flight distance $t$ according to the extinction distribution, which is essential for estimating the Volume Rendering Equation (VRE) [S1, unofficial]. What sets this family apart is its focus on handling heterogeneous media where the extinction coefficient $\sigma_t$ varies spatially, often lacking closed-form cumulative distribution functions (CDFs) [S1, unofficial]. Unlike [Distance Sampling](8978-distance-sampling.md) which provides the general framework, these methods provide specific algorithms to invert the transmittance integral for complex media.

## Shared principles
All methods in this family aim to sample a distance $t$ from the probability density function (PDF) defined by the medium's extinction. The core mathematical principle is inverting the Cumulative Distribution Function (CDF) of the transmittance.

The probability that a photon travels a distance $t$ without interaction is given by the transmittance $T(t)$:
$$ T(t) = \exp\left(-\int_0^t \sigma_t(\mathbf{x} + \omega \tau) d\tau\right) $$

To sample $t$, one generates a uniform random number $u \in [0, 1]$ and solves for $t$ in:
$$ u = 1 - T(t) = 1 - \exp\left(-\int_0^t \sigma_t(\mathbf{x} + \omega \tau) d\tau\right) $$

This requires evaluating the integral of the extinction coefficient along the ray. The methods differ in how they approximate or compute this integral and solve for $t$ [S1, unofficial].

## Subtopics

### [Analytic Sampling](14526-parametric-transmittance-sampling.md)
Uses closed-form solutions for the CDF inverse. Applicable only to simple media (e.g., homogeneous, linearly, or exponentially varying extinction). Unbiased and efficient.

### [Regular Tracking](9223-closed-form-tracking.md)
A semi-analytic approach using 3D DDA or hierarchical voxel grids to sum extinction over piecewise-constant intervals. Unbiased but inefficient if free paths cross many boundaries.

### [Ray Marching](9002-whole-volume-distance-sampling.md)
An approximate method using a constant step size to find the collision distance. It is biased (ignoring thin features) and inaccurate for high-frequency media, but simple to implement for any volume.

### [Rejection Sampling](9576-rejection-sampling-for-arbitrary-media.md)
Uses a bounding volume or majorant function to sample from a simpler distribution and reject samples that do not match the target extinction. Handles arbitrary media but can be inefficient if the majorant is loose.

## Background
The concept of free-path sampling originates from the radiative transfer equation formulation for participating media. In homogeneous media, the flight distance follows an exponential distribution with a closed-form inverse CDF. For heterogeneous media, closed-form solutions are rare, existing only for simple variations like linear or exponential extinction [S1, unofficial].

Historically, three main approaches emerged:
1.  **Analytic**: Direct inversion where possible [S1, unofficial].
2.  **Semi-Analytic (Regular Tracking)**: Using discrete grid structures (like 3D DDA) to compute the integral iteratively [S1, unofficial].
3.  **Approximate (Ray Marching)**: Using fixed steps to approximate the integral, introducing bias [S1, unofficial].

Delta tracking (a form of rejection sampling) is also a key method for arbitrary media, though not explicitly detailed in the provided notes, it falls under the umbrella of free-path sampling for heterogeneous media [S1, unofficial].

## Sources
[S1] Participating media, https://imaging.cs.cmu.edu/15-468/2024_spring/lectures/lecture_12.pdf, fetched 2026-10-29 (unofficial)
