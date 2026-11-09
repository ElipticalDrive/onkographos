# Importance Sampling for Sphere Surface Area

Also called: Sphere Light Shading with Importance Surface Area Sampling, getSampleImportanceSolidAngle

## Overview
Estimates direct illumination from a spherical light by sampling points on its visible surface area, with a probability density function proportional to the projected area of the light as seen from the shading point [S1, unofficial]. It differs from [Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) by explicitly generating points on the sphere's geometry rather than in solid angle space [S1, unofficial].

## How it works
**Inputs:**
*   Shading point $X$ (3D vector) [S1, unofficial].
*   Surface normal at $X$, $\mathbf{n}$ (3D vector, normalized) [S1, unofficial].
*   Sphere light center $C$ (3D vector) [S1, unofficial].
*   Sphere light radius $R$ (scalar) [S1, unofficial].
*   Two uniform random numbers $\eta_1, \eta_2 \in [0, 1)$ [S1, unofficial].

**Outputs:**
*   Sample direction $\mathbf{d}$ (3D vector, normalized) pointing from $X$ to a point on the sphere surface [S1, unofficial].

**Data Structures:**
*   Local coordinate system basis $(\mathbf{U}, \mathbf{V}, \mathbf{W})$ constructed from $\mathbf{n}$ and the vector to the light center [S1, unofficial].

**Steps:**
1.  Compute the vector from the shading point to the light center: $\mathbf{XC} = C - X$ [S1, unofficial].
2.  Compute the squared distance: $d^2 = \|\mathbf{XC}\|^2$ [S1, unofficial].
3.  Construct a local orthonormal basis:
    *   $\mathbf{W} = \mathbf{XC} / \|\mathbf{XC}\|$ (normalized vector to center) [S1, unofficial].
    *   $\mathbf{V} = \mathbf{n} \times \mathbf{W}$, then normalize $\mathbf{V}$ [S1, unofficial].
    *   $\mathbf{U} = \mathbf{V} \times \mathbf{W}$ [S1, unofficial].
4.  Calculate the maximum half-angle $\theta_{max}$ of the sphere as seen from $X$:
    $$ \sin \theta_{max} = \frac{R}{\sqrt{d^2}} $$
    $$ \cos \theta_{max} = \sqrt{1 - \sin^2 \theta_{max}} $$
    [S1, unofficial]
5.  Generate sample angles $\Theta$ and $\Phi$ using $\eta_1$ and $\eta_2$:
    $$ \Theta = \arccos(1 - \eta_1 + \eta_1 \sqrt{1 - \frac{R^2}{d^2}}) $$
    $$ \Phi = 2 \pi \eta_2 $$
    [S1, unofficial]
6.  Construct the sample direction in the local coordinate system:
    $$ \mathbf{d}_{local} = \mathbf{U} (\cos \Phi \sin \Theta) + \mathbf{V} (\sin \Phi \sin \Theta) + \mathbf{W} \cos \Theta $$
    [S1, unofficial]
7.  Normalize $\mathbf{d}_{local}$ and return $\mathbf{d}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_sphere_surface_area(X, normal, C, R, eta1, eta2):
    XC = C - X
    distance2 = dot(XC, XC)
    sin_theta_max = R / sqrt(distance2)
    cos_theta_max = sqrt(1.0 - sin_theta_max * sin_theta_max)
    W = normalize(XC)
    V = normalize(cross(W, normal))
    U = cross(V, W)
    theta = acos(1.0 - eta1 + eta1 * sqrt(1.0 - R * R / distance2))
    phi = 2.0 * PI * eta2
    d_local = U * (cos(phi) * sin(theta)) + V * (sin(phi) * sin(theta)) + W * cos(theta)
    return normalize(d_local), sin_theta_max, cos_theta_max
```
[S1, unofficial]

**Estimator Update:**
When used in a path tracer, the radiance contribution is updated using a geometric area term [S1, unofficial]. The implementation computes the area as:
$$ \text{Area} = 2 \pi (1 - \sin \theta_{max}) \frac{R^2}{d^2} $$
[S1, unofficial]
The radiance estimator updates as:
$$ L_r += f_r L_i (1 - \text{inShadow}) \cos \theta_i \cos \theta_o \text{Area} $$
[S1, unofficial]
This term is multiplied by $f_r L_i (1-\text{inShadow}) \cos \theta_i \cos \theta_o$ [S1, unofficial]. The source code applies this area term directly without an explicit PDF division, implying the PDF is handled implicitly within the geometric formulation [S1, unofficial].

**Edge Cases:**
*   If $X$ is inside the sphere, the method is invalid for external lighting [S1, unofficial].
*   If $d < R$, the point is inside the light [S1, unofficial].
*   Precision issues occur when $d \approx R$ due to grazing angles [S1, unofficial].

## Subtopics

### Importance Surface Area Sampling
This variant uses rejection sampling to generate points on the sphere surface [S1, unofficial]. It loops to generate $\theta = \arccos(1 - 2\eta_1)$ and $\phi = 2\pi\eta_2$ on a unit sphere, scales by $R$, adds center $C$ to get $Y$, computes the normal at $Y$, and checks if $\mathbf{n}_Y \cdot (X - Y) \le \epsilon$ [S1, unofficial]. If the condition fails, it rejects and resamples [S1, unofficial]. The parameter $\epsilon$ is a floating-point threshold for the dot product test [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the analytical method [S1, unofficial]. The rejection-based variant has expected $O(1)$ but can be slower due to rejection loops [S1, unofficial].
*   **Memory Complexity:** $O(1)$ [S1, unofficial].
*   **Strengths:** Intuitive geometric interpretation and direct surface sampling [S1, unofficial].
*   **Limitations:** The rejection sampling variant can be inefficient if the sphere is small or far away [S1, unofficial]. The analytical method relies on a specific geometric area term that bypasses explicit PDF normalization in the provided implementation [S1, unofficial].
*   **When to use:** When the light is a sphere and direct surface sampling is preferred over solid angle sampling for variance reduction in specific BRDFs [S1, unofficial].

## Background
The method is implemented in the `VolumeSingleScattering` ray tracer project as one of several sampling strategies for spherical light sources [S1, unofficial]. It is grouped alongside Importance Solid Angle Sampling, Uniform Solid Angle Sampling, and Cosine-weighted Hemispherical Importance Sampling [S1, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
