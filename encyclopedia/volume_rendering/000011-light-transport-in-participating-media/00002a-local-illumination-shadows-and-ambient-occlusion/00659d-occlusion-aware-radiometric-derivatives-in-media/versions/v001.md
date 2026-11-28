# Occlusion-Aware Radiometric Derivatives in Media

Also called: Concentric triangulations, Piecewise linear radiance representation

## Overview
This method formulates radiance at a point $\mathbf{x}$ within a participating medium as a piecewise linear representation of incoming radiance, enabling the computation of first- and second-order derivatives for radiance caching [S1]. It extends surface-based radiometric derivative techniques to handle light transport from both surfaces and participating media by using concentric triangulations and geometry terms that account for transmittance [S1]. Unlike methods that only handle surface occlusion, this approach models visibility changes as changes in the 3D geometry term between triangle points and the observer, allowing for unified derivative calculation in volumetric scenes [S1].

## How it works
**Inputs:**
*   A query point $\mathbf{x}$ in a participating medium.
*   A set of stochastic angular samples $\mathbf{y}_s$ distributed around $\mathbf{x}$.
*   Scene geometry (surfaces) and medium density distribution $\sigma_t(\mathbf{x})$.

**Outputs:**
*   Approximated radiance $L(\mathbf{x}, \omega_o)$ and its derivatives.

**Data Structures:**
*   **Concentric Rings ($R$):** A set of rings at increasing distances $r_i$ from $\mathbf{x}$. Each ring $r_i$ contains a set of segments (in 2D) or triangles (in 3D) $\mathcal{L}_i$.
*   **Sample Points ($\mathbf{y}$):** Vertices of the triangulation. For surfaces, $\mathbf{y}$ is the first intersection point. For media, $\mathbf{y}$ lies on the ring at distance $r_i$.
*   **Geometry Term ($G$):** A scalar value associated with each segment/triangle, defined as $G = \frac{\cos \theta_y}{\|\mathbf{x}\mathbf{y}\|}$ in 2D, where $\theta_y$ is the projected angle of the segment towards $\mathbf{x}$ and $\|\mathbf{x}\mathbf{y}\|$ is the distance [S1]. In 3D, this generalizes to the standard geometry term between the triangle and $\mathbf{x}$.

**Steps:**
1.  **Angular Sampling:** Generate a set of stochastic directions around $\mathbf{x}$.
2.  **Ring Construction:** Define concentric rings $r_i$ at increasing distances. The outermost ring $r_s$ has vertices on surface intersections [S1].
3.  **Sample Placement:**
    *   For each direction, find the first surface intersection $\mathbf{y}_{surf}$.
    *   If $\|\mathbf{x}\mathbf{y}_{surf}\| < r_i$, place a zero-radiance sample at $\mathbf{y}_{surf}$ for all rings $r_i$ beyond this distance [S1].
    *   Otherwise, place a sample at distance $r_i$ along the ray. The radiance value at this sample is determined by the outgoing radiance from the medium at that distance.
4.  **Triangulation:** Connect adjacent samples within each ring $r_i$ to form segments (2D) or triangles (3D) $\mathcal{L}_i$ [S1].
5.  **Geometry Term Calculation:** Compute the geometry term $G$ for each segment/triangle. In 2D, $G = \frac{\cos \theta_y}{\|\mathbf{x}\mathbf{y}\|}$, where $\theta_y$ is the angle between the segment and the view direction [S1]. This term implicitly encodes visibility changes.
6.  **Radiance Approximation:** Approximate the radiance $L(\mathbf{x}, \omega_o)$ by summing contributions from all rings and segments:
    $$ L(\mathbf{x}, \omega_o) \approx \sum_{r_i \in R} \sum_{\ell_j \in \mathcal{L}_i} \frac{L_j(\mathbf{x}, \omega_o)}{\text{pdf}(r_i)} $$
    where $\text{pdf}(r_i)$ is the probability of sampling distance $r_i$ [S1].
7.  **Derivative Computation:** Compute derivatives of radiance with respect to $\mathbf{x}$ by differentiating the geometry term $G$ and the radiance values $L_j$. Changes in $G$ due to translation of $\mathbf{x}$ model changes in observed radiance [S1].

**Parameters:**
*   **Number of Rings:** Not given in the sources.
*   **Angular Resolution:** Determined by the number of stochastic samples. Not given in the sources.
*   **Distance Sampling PDF:** $\text{pdf}(r_i)$ depends on the sampling strategy. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Occlusion:** If occluding geometry exists closer than a ring distance $r_i$, the sample at $r_i$ must be zeroed out [S1]. Failure to do so results in incorrect radiance and derivatives.
*   **Grazing Angles:** In 2D, occlusions generate segments at grazing angles, where derivatives are proportional to the steepness of the segment [S1]. Numerical stability may be an issue here.
*   **Transmittance:** The method requires accounting for transmittance in the geometry term for media, which differs from the surface-only case [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Unified formulation for surface and media light transport; captures occlusion-aware derivatives; suitable for radiance caching [S1].
*   **Limitations:** Requires stochastic sampling; accuracy depends on the number of rings and angular samples; complex to implement compared to simple shadow rays [S1].
*   **When to Use:** When high-quality volumetric lighting with accurate shadows and derivatives is needed, particularly for radiance caching or real-time applications where precomputation is feasible [S1].
*   **When Not to Use:** For simple volumetric effects without self-shadowing or derivatives; when computational resources are extremely limited [S1].

## Background
This method is an extension of the work by Schwarzhaupt et al. [2012] on global illumination on surfaces, adapted to handle participating media [S1]. It was introduced in the paper "Second-Order Occlusion-Aware Volumetric Radiance Caching" [S1]. The key insight is that the geometry term of a piecewise linear representation implicitly encodes visibility changes, allowing for efficient derivative computation [S1].

## Sources
[S1] Second-Order Occlusion-Aware Volumetric Radiance Caching, https://graphics.unizar.es/papers/2018TOG_SecondOrderMediaCaching.pdf, fetched 2026-10-29
