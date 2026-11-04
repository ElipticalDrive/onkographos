# Importance Sampling for Sphere Surface Area

Also called: Sphere Light Shading with Importance Surface Area Sampling, getSampleImportanceSolidAngle

## Overview
Estimates direct illumination from a spherical light by sampling points on its visible surface area, with a probability density function (PDF) proportional to the projected area of the light as seen from the shading point [S1, unofficial]. It differs from [Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) by explicitly generating points on the sphere's geometry rather than in solid angle space, which simplifies the calculation of the geometric term but requires rejection sampling or careful coordinate transformation to ensure uniform coverage of the visible cap [S1, unofficial].

## How it works
**Inputs:**
*   Shading point $X$ (3D vector).
*   Surface normal at $X$, $\mathbf{n}$ (3D vector, normalized).
*   Sphere light center $C$ (3D vector).
*   Sphere light radius $R$ (scalar).
*   Two uniform random numbers $\eta_1, \eta_2 \in [0, 1)$.

**Outputs:**
*   Sample direction $\mathbf{d}$ (3D vector, normalized) pointing from $X$ to a point on the sphere surface.

**Data Structures:**
*   Local coordinate system basis $(\mathbf{U}, \mathbf{V}, \mathbf{W})$ constructed from $\mathbf{n}$ and the vector to the light center.

**Steps:**
1.  Compute the vector from the shading point to the light center: $\mathbf{XC} = C - X$.
2.  Compute the squared distance: $d^2 = \|\mathbf{XC}\|^2$.
3.  Construct a local orthonormal basis:
    *   $\mathbf{W} = \mathbf{XC} / \|\mathbf{XC}\|$ (normalized vector to center).
    *   $\mathbf{V} = \mathbf{n} \times \mathbf{W}$, then normalize $\mathbf{V}$. If $\mathbf{n}$ is parallel to $\mathbf{XC}$, $\mathbf{V}$ is undefined; [G] use an arbitrary perpendicular vector (e.g., if $|\mathbf{n} \cdot \mathbf{W}| > 0.999$, set $\mathbf{V} = (1,0,0) \times \mathbf{W}$, else $\mathbf{V} = \mathbf{n} \times \mathbf{W}$).
    *   $\mathbf{U} = \mathbf{V} \times \mathbf{W}$.
4.  Calculate the maximum half-angle $\theta_{max}$ of the sphere as seen from $X$:
    $$ \sin \theta_{max} = \frac{R}{\sqrt{d^2}} $$
    $$ \cos \theta_{max} = \sqrt{1 - \sin^2 \theta_{max}} $$
5.  Generate sample angles $\Theta$ and $\Phi$ using $\eta_1$ and $\eta_2$:
    $$ \Theta = \arccos(1 - \eta_1 + \eta_1 \sqrt{1 - \frac{R^2}{d^2}}) $$
    $$ \Phi = 2 \pi \eta_2 $$
    *Note: This formula maps uniform random variables to a distribution proportional to the projected area of the spherical cap.*
6.  Construct the sample direction in the local coordinate system:
    $$ \mathbf{d}_{local} = \mathbf{U} (\cos \Phi \sin \Theta) + \mathbf{V} (\sin \Phi \sin \Theta) + \mathbf{W} \cos \Theta $$
7.  Normalize $\mathbf{d}_{local}$ and return $\mathbf{d}$.

**Estimator Update:**
When used in a path tracer, the radiance contribution is updated as:
$$ L_r += f_r(\mathbf{d}, \mathbf{v}) L_i \frac{1 - \text{inShadow}}{\text{PDF}(\mathbf{d})} $$
Where the PDF for importance surface area sampling is derived from the area distribution. The source code snippet [S1, unofficial] uses a simplified geometric area term in the estimator:
$$ \text{Area} = 2 \pi (1 - \sin \theta_{max}) \frac{R^2}{d^2} $$
$$ L_r += f_r L_i (1 - \text{inShadow}) \cos \theta_i \cos \theta_o \text{Area} $$
This implies the PDF is handled implicitly or the term represents the solid angle approximation. For strict Monte Carlo correctness, the PDF should be $p(\omega) = \frac{\cos \theta}{\pi (1 - \cos \theta_{max})}$ for uniform solid angle, but for surface area importance, the PDF is $p(\omega) = \frac{\sin \theta_{max} \cos \theta}{\pi (1 - \cos \theta_{max})}$? No, the source [S1, unofficial] code snippet suggests a specific implementation where the area term is multiplied directly. The standard importance surface area PDF for a sphere is $p(\omega) = \frac{R^2 \cos \theta}{d^3 (1 - \cos \theta_{max})}$? The source does not explicitly define the PDF function, only the area term used in the estimator. [G] In standard implementations, if sampling the surface area uniformly, the PDF is $1 / A_{visible}$. The projected area is $\pi R^2 / d^2$? No. The source code uses `Area = 2 * M_PI * (1 - sinThetaMax) * lightR * lightR / distance2`. This looks like an approximation of the solid angle $\Omega = 2\pi(1-\cos\theta_{max})$. The formula in the notes says `Area = 2π(1 - sinθ_max)`. This is likely a typo in the source for `cos`. However, we must follow the source. The source formula is:
$$ \text{Area} = 2 \pi (1 - \sin \theta_{max}) \frac{R^2}{d^2} $$
This term is multiplied by $f_r L_i (1-\text{inShadow}) \cos \theta_i \cos \theta_o$.

**Edge Cases:**
*   If $X$ is inside the sphere, the method is invalid for external lighting.
*   If $d < R$, the point is inside the light.
*   Precision issues when $d \approx R$ (grazing angle).

## Subtopics

### Importance Surface Area Sampling
A variant that explicitly loops to generate a point $Y$ on the sphere surface and checks if it faces the shading point $X$. It generates $\theta = \arccos(1 - 2\eta_1)$ and $\phi = 2\pi\eta_2$ on a unit sphere, scales by $R$, adds center $C$ to get $Y$, computes the normal at $Y$, and checks if $\mathbf{n}_Y \cdot (X - Y) \le \epsilon$. If not, it rejects and resamples. This ensures the sampled point is on the visible hemisphere of the sphere relative to $X$. Parameter $\epsilon$ is a floating-point threshold for the dot product test [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the analytical method. The rejection-based variant has expected $O(1)$ but can be slower due to rejection loops.
*   **Memory Complexity:** $O(1)$.
*   **Strengths:** Intuitive geometric interpretation; samples the light surface directly.
*   **Limitations:** The rejection sampling variant can be inefficient if the sphere is small or far away (though the analytical method avoids this). The analytical method's PDF derivation is complex and source [S1, unofficial] uses a specific area term that may not correspond to a standard PDF normalization without further context.
*   **When to use:** When the light is a sphere and direct surface sampling is preferred over solid angle sampling for variance reduction in specific BRDFs.

## Background
Originated in the context of ray tracing implementations for participating media and global illumination, specifically in the `VolumeSingleScattering` project [S1, unofficial]. It is one of several sampling strategies (alongside Importance Solid Angle, Uniform Solid Angle, etc.) implemented to handle spherical light sources efficiently.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
