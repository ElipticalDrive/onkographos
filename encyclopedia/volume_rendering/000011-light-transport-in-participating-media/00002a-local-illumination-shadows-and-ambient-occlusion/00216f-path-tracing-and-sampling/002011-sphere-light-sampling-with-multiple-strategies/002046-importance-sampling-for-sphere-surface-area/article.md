# Importance Sampling for Sphere Surface Area

Also called: Sphere Light Shading with Importance Surface Area Sampling, getSampleImportanceSolidAngle

## Overview
Estimates direct illumination from a spherical light source by sampling points on its visible surface area, with a probability density function (PDF) proportional to the projected area [S1, unofficial]. Unlike [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) which samples the solid angle subtended by the light, this method samples the physical surface points directly, requiring a geometric correction factor in the estimator to account for the varying projected area relative to the viewing direction [S1, unofficial].

## How it works
**Inputs:** Intersection point $X$ on the shading surface, surface normal $N$ at $X$, sphere light center $C$, sphere light radius $R$, and two uniform random numbers $\eta_1, \eta_2 \in [0, 1)$ [S1, unofficial].
**Outputs:** Sample direction $D$ from $X$ to a point on the sphere surface, and the updated radiance estimator $L_r$ [S1, unofficial].
**Data Structures:** Vectors $U, V, W$ form an orthonormal basis aligned with the local geometry, and scalar `distance2XC` stores the squared Euclidean distance between $X$ and $C$ [S1, unofficial].

**Steps:**
1. **Compute Geometry:** Calculate the vector from the intersection point to the light center: $\vec{XC} = C - X$. Compute the squared distance: $\text{distance2XC} = \|\vec{XC}\|^2$. Compute the half-angle sine and cosine of the light cone: $\sin\theta_{\max} = R / \|\vec{XC}\|$, $\cos\theta_{\max} = \sqrt{1 - \sin^2\theta_{\max}}$ [S1, unofficial].
2. **Construct Local Basis:** Define the local coordinate system basis vectors: $W = \vec{XC} / \|\vec{XC}\|$, $V = (W \times N) / \|W \times N\|$, $U = V \times W$ [S1, unofficial]. [G] If $W$ and $N$ are parallel, choose an arbitrary orthogonal vector for $V$ to avoid division by zero.
3. **Sample Angles:** Generate two uniform random numbers $\eta_1, \eta_2 \in [0, 1)$ [S1, unofficial]. Compute the polar angle $\Theta$ and azimuthal angle $\Phi$: $\Theta = \arccos(1 - \eta_1 + \eta_1 \sqrt{1 - R^2 / \text{distance2XC}})$, $\Phi = 2\pi \eta_2$ [S1, unofficial].
4. **Construct Direction:** Construct the sample direction in the local coordinate system: $D_{\text{local}} = U (\cos\Phi \sin\Theta) + V (\sin\Phi \sin\Theta) + W \cos\Theta$. Normalize and return $D = D_{\text{local}} / \|D_{\text{local}}\|$ [S1, unofficial].
5. **Estimator Update:** Trace a ray from $X$ in direction $D$ to determine shadow status $(1 - \text{inShadow})$ [S1, unofficial]. Calculate the projected area factor: $\text{Area} = 2\pi (1 - \sin\theta_{\max}) R^2 / \text{distance2XC}$ [S1, unofficial]. Update the radiance estimator: $L_r += f_r L_i (1 - \text{inShadow}) \cos\theta_i \cos\theta_o \cdot \text{Area}$ [S1, unofficial]. Average over $N$ samples [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

PI = math.pi

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def normalize(v):
    l = math.sqrt(dot(v, v))
    return (v[0]/l, v[1]/l, v[2]/l)

def sample_sphere_surface_area(X, normal, light_center, light_radius, eta1, eta2):
    XC = (light_center[0]-X[0], light_center[1]-X[1], light_center[2]-X[2])
    distance2XC = dot(XC, XC)
    dist = math.sqrt(distance2XC)

    sin_theta_max = light_radius / dist

    W = normalize(XC)
    cross_WN = cross(W, normal)
    len_cross_WN = math.sqrt(dot(cross_WN, cross_WN))
    if len_cross_WN < 1e-6:
        # Degenerate case: W parallel to normal. Choose arbitrary orthogonal vector.
        if abs(W[0]) < 0.9:
            V = normalize(cross(W, (1.0, 0.0, 0.0)))
        else:
            V = normalize(cross(W, (0.0, 1.0, 0.0)))
    else:
        V = (cross_WN[0]/len_cross_WN, cross_WN[1]/len_cross_WN, cross_WN[2]/len_cross_WN)
    U = cross(V, W)

    term = math.sqrt(1.0 - (light_radius * light_radius) / distance2XC)
    cos_theta = 1.0 - eta1 + eta1 * term
    cos_theta = max(-1.0, min(1.0, cos_theta))
    theta = math.acos(cos_theta)
    phi = 2.0 * PI * eta2

    sin_theta = math.sin(theta)
    cos_phi = math.cos(phi)
    sin_phi = math.sin(phi)

    D_local = (
        U[0] * (cos_phi * sin_theta) + V[0] * (sin_phi * sin_theta) + W[0] * math.cos(theta),
        U[1] * (cos_phi * sin_theta) + V[1] * (sin_phi * sin_theta) + W[1] * math.cos(theta),
        U[2] * (cos_phi * sin_theta) + V[2] * (sin_phi * sin_theta) + W[2] * math.cos(theta)
    )
    D = normalize(D_local)
    return D

def update_estimator(X, normal, D, light_center, light_radius, fr, Li, in_shadow, cos_theta_o):
    XC = (light_center[0]-X[0], light_center[1]-X[1], light_center[2]-X[2])
    distance2XC = dot(XC, XC)
    dist = math.sqrt(distance2XC)
    sin_theta_max = light_radius / dist

    Area = 2.0 * PI * (1.0 - sin_theta_max) * (light_radius * light_radius) / distance2XC
    cos_theta_i = dot(normal, D)

    contribution = fr * Li * (1.0 - in_shadow) * cos_theta_i * cos_theta_o * Area
    return contribution
```

## Subtopics

### Importance Surface Area Sampling
A variant that explicitly loops to generate a point $Y$ on the sphere surface and checks visibility [S1, unofficial]. It generates $\eta_1, \eta_2$, computes $\theta = \arccos(1 - 2\eta_1)$ and $\phi = 2\pi\eta_2$, then constructs $Y = (R\sin\theta\cos\phi, R\sin\theta\sin\phi, R\cos\theta) + C$ [S1, unofficial]. It computes the normal $C_Y$ at $Y$ and the vector $YX = X - Y$ [S1, unofficial]. If $C_Y \cdot YX \le \epsilon$, the sample is rejected and the loop repeats [S1, unofficial]. This rejection sampling approach ensures the sampled point faces the shading point $X$ [S1, unofficial]. Parameter $\epsilon$ is a floating-point threshold for the dot product test [S1, unofficial].

### [Importance Sampling](9521-importance-sampling.md)
A general variance reduction technique for Monte Carlo integration that accelerates convergence by selecting samples from a PDF that mimics the integrand shape. It sets itself apart by providing the theoretical framework, while this method applies it specifically to spherical surface area sampling.

### [Occlusion-Aware vMF Importance Sampler](9557-occlusion-aware-vmf-importance-sampler.md)
A lightweight importance sampler for incoming radiance using a Neural Graphics Primitive to predict a mixture of von Mises-Fisher distributions. It solves the problem of approximating true incoming light distributions including occlusions by allowing the model to turn off lobes for occluded sources. It builds on this method by adding spatial variation and occlusion awareness via the NGP.

## Operating characteristics
Not given in the sources.

## Background
It is one of several sampling strategies (alongside Importance Solid Angle, Uniform Solid Angle, etc.) implemented to estimate direct illumination in participating media and surface rendering [S1, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
