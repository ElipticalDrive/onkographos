# Change of Variables for Volume Rendering Integral
Also called: Change of Variables: Volume, Surface Integral Formulation of VRI, Volume Rendering Integral Change of Variables

## Overview
This method transforms the volume rendering integral from a parameterization over ray direction and distance into a direct integration over the 3D spatial volume. It replaces directional queries with direct spatial coordinates, allowing the radiance contribution of all points in the participating medium to be summed directly rather than sampled along rays. This formulation is the volumetric analog of the surface integral change of variables, enabling path integral formulations where the integration domain is the volume $\mathcal{V}$ rather than the solid angle $\mathcal{S}^2$ and distance $r$.

## How it works
The transformation relies on the Jacobian of the coordinate change from spherical coordinates (direction $\bo$ and distance $r$) to Cartesian coordinates (position $\bx'$).

**Inputs and Outputs:**
- Input: A view point $\bx$, a function $f(\bx')$ representing the integrand (e.g., emitted and scattered light attenuated by transmittance), and the visibility function $V(\bx' \to \bx)$.
- Output: The integral value over the volume $\mathcal{V}$.

**Data Structures:**
- $\bx$: View point position (3D vector).
- $\bx'$: Integration variable position (3D vector).
- $\mathcal{V}$: The volume domain of integration.

**Formula:**
The change of variables for the volume rendering integral is given by [S1, unofficial]:

$$
\int_{\mathcal{S}^2} \int_0^{\|\raycast(\bx, \bo) - \bx\|} f(\bx + r\bo) \, dr \, d\bo = \int_{\mathcal{V}} \frac{1}{\|\bx' - \bx\|^2} V(\bx' \to \bx) f(\bx') \, dV(\bx')
$$

**Symbol Definitions:**
- $\mathcal{S}^2$: The unit sphere of directions.
- $\bx$: The camera or sensor position.
- $\bo$: A unit direction vector on $\mathcal{S}^2$.
- $r$: Distance along the ray from $\bx$.
- $\raycast(\bx, \bo)$: The intersection point of the ray starting at $\bx$ in direction $\bo$ with the boundary of the volume.
- $f(\bx')$: The integrand evaluated at position $\bx'$. In the context of the volume rendering equation, this typically includes emission and scattering terms attenuated by transmittance from $\bx'$ to $\bx$.
- $\mathcal{V}$: The volume of the participating medium.
- $\|\bx' - \bx\|$: The Euclidean distance between the integration point $\bx'$ and the view point $\bx$.
- $V(\bx' \to \bx)$: The visibility function, which is 1 if the path from $\bx'$ to $\bx$ is unoccluded within the medium, and 0 otherwise.
- $dV(\bx')$: The differential volume element at $\bx'$.

**Derivation Logic:**
1.  Start with the integral over solid angle $d\bo$ and distance $dr$.
2.  The position $\bx'$ is defined as $\bx + r\bo$.
3.  The differential volume element in spherical coordinates centered at $\bx$ is $dV = r^2 dr d\bo = \|\bx' - \bx\|^2 dr d\bo$.
4.  Therefore, $dr d\bo = \frac{1}{\|\bx' - \bx\|^2} dV(\bx')$.
5.  Substitute this into the integral. The limits of integration for $r$ (from 0 to the boundary) become the extent of the volume $\mathcal{V}$ along the ray, which is handled by the visibility function $V(\bx' \to \bx)$ and the domain $\mathcal{V}$.

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Singularity at $\bx' = \bx$**: The term $\frac{1}{\|\bx' - \bx\|^2}$ becomes singular when the integration point coincides with the view point. In practice, this is handled by the behavior of $f(\bx')$ (which may include transmittance $T(\bx \to \bx')$ that goes to 1, but emission/scattering terms must be well-behaved) or by excluding a small neighborhood around $\bx$.
- **Visibility Calculation**: Computing $V(\bx' \to \bx)$ for every point in the volume is computationally expensive and may require additional ray casting or precomputed visibility data.
- **Boundary Handling**: The original integral limits $r$ to the boundary of the volume. The volume integral implicitly handles this via the domain $\mathcal{V}$ and the visibility function.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def change_of_variables_volume_integral(view_point, volume_domain, integrand_func, visibility_func):
    """
    Computes the volume rendering integral using the change of variables formulation.
    
    Args:
        view_point (Vector3): The camera/sensor position bx.
        volume_domain (Volume): The 3D volume V to integrate over.
        integrand_func (Function): A function f(bx_prime) that returns the 
                                   unattenuated radiance contribution at bx_prime.
        visibility_func (Function): A function V(bx_prime, bx) that returns 1 if 
                                    bx_prime is visible from bx, 0 otherwise.
    
    Returns:
        float: The value of the integral.
    """
    # This is a conceptual representation. Actual implementation requires
    # a numerical integration method (e.g., Monte Carlo, quadrature) over the volume.
    
    total_integral = 0.0
    
    # Iterate over volume samples (e.g., via Monte Carlo sampling)
    for bx_prime in volume_domain.sample():
        distance_sq = squared_length(bx_prime - view_point)
        
        # Avoid division by zero if sample coincides with view point
        if distance_sq < epsilon:
            continue
            
        weight = 1.0 / distance_sq
        visibility = visibility_func(bx_prime, view_point)
        f_value = integrand_func(bx_prime)
        
        # Estimate integral contribution
        # In Monte Carlo, this would involve importance sampling and PDF evaluation
        contribution = weight * visibility * f_value
        total_integral += contribution
        
    return total_integral
```

## Subtopics

### Change of Variables for Surface Integration
This is the surface analog of the volume change of variables. It transforms the integral over solid angle $\mathcal{S}^2$ to an integral over surface area $\mathcal{M}$ [S1, unofficial]. It replaces directional queries with direct spatial coordinates on the surface.
**Formula:**
$$
\int_{\mathcal{S}^2} f(\raycast(\bx, \bo)) \, d\bo = \int_{\mathcal{M}} f(\bx') \frac{|\vec{n}' \cdot \bo_{\bx \bx'}|}{\|\bx' - \bx\|^2} V(\bx' \to \bx) \, dA(\bx')
$$
**Parameters:**
- $V(\bx' \to \bx)$: Visibility function (1 if visible, 0 if occluded) [S1, unofficial].
- $\bo_{\bx \bx'}$: Direction from $\bx$ to $\bx'$ [S1, unofficial].
- $\vec{n}'$: Normal vector at $\bx'$ [S1, unofficial].
- $dA(\bx')$: Differential area element at $\bx'$ [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the numerical integration method used over the volume. For Monte Carlo, it scales with the number of samples. The evaluation of the visibility function $V(\bx' \to \bx)$ can be a significant bottleneck if not precomputed or approximated.
- **Memory Complexity**: Depends on the representation of the volume $\mathcal{V}$ and the visibility function.
- **Strengths**: Provides a unified framework for path integral formulations, treating volume and surface contributions similarly. It can simplify the analysis of light transport in participating media by removing the explicit ray parameterization.
- **Limitations**: The $1/\|\bx' - \bx\|^2$ term can lead to high variance in Monte Carlo integration if not sampled carefully. The visibility function is difficult to compute efficiently for complex scenes.
- **When to use**: When formulating path integrals for participating media or when a direct spatial integration approach is preferred over ray marching.
- **When not to use**: For simple homogeneous volumes where analytical solutions exist, or when real-time performance is critical and ray marching is more efficient.

## Background
This formulation is derived from the principles of light transport in participating media and is used in the context of the volume rendering equation. It is presented in the notes on the Volume Rendering Equation 2, which discusses the measurement equation and the path integral formulation of light transport [S1, unofficial]. The concept is analogous to the surface integral formulation widely used in path tracing and bidirectional path tracing.

## Sources
[S1] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
