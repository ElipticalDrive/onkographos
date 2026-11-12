# Occupancy-Opacity Factorization

Also called: factorized opacity, sigma-alpha factorization

## Overview
Occupancy-Opacity Factorization decouples a Gaussian primitive's geometric presence from its optical properties by splitting the single standard opacity parameter into two learnable attributes: geometric occupancy ($\sigma$) and optical opacity ($\alpha$) [S1, unofficial]. This solves the problem where a single opacity parameter cannot simultaneously model a surface that is geometrically solid (for sharp reflections) yet optically clear (for light transmission) [S1, unofficial]. It sets itself apart from siblings like [Textured-GS](9436-textured-gs.md) and [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md) by explicitly modeling the probabilistic interaction of rays with the Gaussian substance, enabling deferred shading for high-frequency reflection modeling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians, each defined by position, covariance (geometry), color, and the new parameters $\sigma$ and $\alpha$.
*   **Output:** Rendered color and surface attributes (e.g., normals, roughness) for each pixel.

**Data Structures:**
Each Gaussian $i$ stores:
*   $\sigma_i \in [0, 1]$: Geometric occupancy.
*   $\alpha_i \in [0, 1]$: Optical opacity.
*   $\mathcal{G}_i$: The standard Gaussian weight (spatial falloff) [S1, unofficial].

**Formulas:**
The effective opacity used for volumetric compositing is the product:
$$ \alpha_{\text{eff}, i} = \sigma_i \alpha_i $$
Optical attenuation only occurs where the Gaussian is geometrically present [S1, unofficial].

The probability $p_i$ that the $i$-th Gaussian is the first surface element with which a ray interacts is:
$$ p_i = \sigma_i \mathcal{G}_i \prod_{j=1}^{i-1} (1 - \sigma_j \mathcal{G}_j) $$
Here, $\mathcal{G}_i$ represents the Gaussian weight [S1, unofficial].

The expected value of a surface attribute $\mathbf{a}$ (e.g., normal or roughness) for the first-hit surface is computed as:
$$ \mathbf{A} = \sum_i p_i \mathbf{a}_i $$
This formulation treats the collection of Gaussians as a unified, probabilistic representation of a single surface, justifying deferred shading [S1, unofficial].

**Steps:**
1.  For each ray, sort intersecting Gaussians by depth.
2.  Compute the cumulative probability of non-interaction up to Gaussian $i-1$: $\prod_{j=1}^{i-1} (1 - \sigma_j \mathcal{G}_j)$.
3.  Compute the interaction probability $p_i$ for Gaussian $i$ using the formula above.
4.  Accumulate surface attributes $\mathbf{A}$ weighted by $p_i$.
5.  Composite color using standard alpha blending with $\alpha_{\text{eff}, i} = \sigma_i \alpha_i$.
6.  Apply transparent mask regularization to $\alpha$ during optimization [S1, unofficial].

**Parameters:**
*   $\sigma$: Learnable, constrained to $[0, 1]$.
*   $\alpha$: Learnable, constrained to $[0, 1]$.

**Edge Cases and Pitfalls:**
Gaussians with high geometric occupancy but near-zero optical opacity can exist anywhere in the scene without affecting the final rendered color, leading to unconstrained "ghost" geometries that corrupt the surface representation [S1, unofficial]. This is resolved by a transparent mask regularization loss [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Similar to standard 3D Gaussian Splatting, as the compositing loop remains the same, but with an additional multiplication for $\alpha_{\text{eff}}$ and probability calculation for surface attributes.
*   **Memory Complexity:** Increases by one scalar ($\sigma$) per Gaussian, as $\alpha$ replaces the single opacity scalar.
*   **Strengths:** Enables sharp specular reflections on transparent objects; physically grounded surface representation.
*   **Limitations:** Requires additional regularization to prevent ghost geometries; optimization may be unstable in diffuse regions without strong specular cues [S1, unofficial].
*   **When to use:** When rendering transparent objects with sharp reflections or when surface attributes (normals, roughness) are needed for deferred shading.
*   **When not to use:** For purely opaque scenes where standard opacity suffices, or when computational budget is extremely tight.

## Background
Originated in "RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting" [S1, unofficial]. It addresses the limitation of standard 3D Gaussian Splatting where a single opacity parameter fails to model both solid geometry and optical transparency simultaneously [S1, unofficial].

## Sources
[S1] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
