# Depth Cueing via Linear Attenuation

Also called: Approximate exponential attenuation, Linear attenuation shading model, X-rays with depth cueing

## Overview
This method approximates the perceptual depth cues of exponential attenuation in volume rendering by replacing the variable absorption coefficient with a linear function of position, enabling the integrand to be evaluated as a Fourier projection [S1, unofficial]. It sets itself apart from standard exponential attenuation models by allowing efficient computation via the Fourier Projection-Slice Theorem [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volume density function $\rho(x,y,z)$, volume color function $C(x,y,z)$, constant opacity/absorption coefficient $\tau$, viewing direction vector $\mathbf{V}$, ray segment bounds $a$ and $b$, linear depth constants $m$ and $\mathbf{b}$, ray origin, and ray direction [S1, unofficial].
- Output: Total energy $I_P$ arriving at the end of the viewing ray [S1, unofficial].

**Data Structures:**
- $\rho$: Scalar field representing density [S1, unofficial].
- $C$: Vector field representing color/emission [S1, unofficial].
- $\mathbf{V}$: Normalized 3D vector pointing away from the eye [S1, unofficial].
- $\mathbf{D}(\mathbf{P})$: Linear position function $\mathbf{D} = m\mathbf{P} + \mathbf{b}$, where $m$ is a scalar and $\mathbf{b}$ is a 3D vector constant defining the depth cueing geometry [S1, unofficial].

**Steps:**
1. Define the viewing ray $\mathbf{P}(t)$ parameterized by length $t$ [S1, unofficial].
2. Construct the linear depth function $\mathbf{D}(\mathbf{P}(t)) = m\mathbf{P}(t) + \mathbf{b}$ [S1, unofficial].
3. Compute the dot product $\mathbf{D}(\mathbf{P}(t)) \cdot \mathbf{V}$ to determine the local attenuation weight [S1, unofficial].
4. Evaluate the integrand: $\rho(\mathbf{P}(t)) C(\mathbf{P}(t)) [\frac{\tau}{2} + \frac{\tau}{2} (\mathbf{D}(\mathbf{P}(t)) \cdot \mathbf{V})]$ [S1, unofficial].
5. Integrate the result from $t=a$ to $t=b$ [S1, unofficial].

**Formula:**
The total energy $I_P$ arriving at point $\mathbf{P}(b)$ is given by [S1, unofficial]:

$$
I_P = \int_{a}^{b} \rho(\mathbf{P}(t)) C(\mathbf{P}(t)) \left[ \frac{\tau}{2} + \frac{\tau}{2} (\mathbf{D}(\mathbf{P}(t)) \cdot \mathbf{V}) \right] dt
$$

Where:
- $\rho(\mathbf{P}(t))$: Density at position $\mathbf{P}(t)$ [S1, unofficial].
- $C(\mathbf{P}(t))$: Color/emission at position $\mathbf{P}(t)$ [S1, unofficial].
- $\tau$: Constant opacity/absorption coefficient [S1, unofficial].
- $\mathbf{D}(\mathbf{P}(t))$: Linear function of position, $\mathbf{D} = m\mathbf{P} + \mathbf{b}$ [S1, unofficial].
- $\mathbf{V}$: Normalized viewing direction vector (points away from the eye) [S1, unofficial].
- $a, b$: Start and end parameters of the ray segment [S1, unofficial].
- Note: The constant vector $\mathbf{b}$ in the depth function is distinct from the scalar ray endpoint parameter $b$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_linear_attenuation_energy(rho, color, tau, m, b_vec, v, ray_origin, ray_direction, t_start, t_end, num_samples):
    """
    Computes the integrated energy for linear depth cueing.
    
    Args:
        rho: Function returning density scalar at (x,y,z).
        color: Function returning color vector at (x,y,z).
        tau: Constant opacity coefficient.
        m: Scalar constant for linear depth function.
        b_vec: 3D vector constant for linear depth function (distinct from scalar limit b).
        v: Normalized 3D viewing direction vector (away from eye).
        ray_origin: 3D vector for ray origin.
        ray_direction: 3D vector for ray direction.
        t_start: Start parameter 'a'.
        t_end: End parameter 'b'.
        num_samples: Number of integration steps.
        
    Returns:
        Total energy I_P.
    """
    dt = (t_end - t_start) / num_samples
    total_energy = 0.0
    for i in range(num_samples):
        t = t_start + i * dt
        p_t = (ray_origin[0] + t * ray_direction[0],
               ray_origin[1] + t * ray_direction[1],
               ray_origin[2] + t * ray_direction[2])
        d_t = (m * p_t[0] + b_vec[0],
               m * p_t[1] + b_vec[1],
               m * p_t[2] + b_vec[2])
        dot_product = d_t[0] * v[0] + d_t[1] * v[1] + d_t[2] * v[2]
        weight = (tau / 2.0) + (tau / 2.0) * dot_product
        density = rho(p_t)
        c = color(p_t)
        energy_sample = density * c * weight
        total_energy += energy_sample * dt
    return total_energy
```

**Edge Cases:**
- If $\mathbf{D} \cdot \mathbf{V}$ results in negative weights, the physical interpretation of attenuation may break down, though the formula remains mathematically valid [G].
- Precision issues may arise if $\tau$ is very large or very small relative to the integration step size [G].

## Subtopics

### [Linear Depth Cueing via Fourier Projection-Slice Theorem](1129-linear-depth-cueing-via-fourier-projection-slice-theorem.md)
A shading model that enhances depth perception in X-ray projections by weighting volume points based on their distance from the viewer, without introducing occlusion. It replaces per-voxel shading followed by projection with projection followed by per-pixel shading, allowing depth cueing (linear attenuation) to be computed efficiently by factoring the viewing direction out of the integral. Unlike opaque volume renderings, this method produces X-ray-like projections with no occlusion, but the linear shading provides sufficient shape cues for spatial understanding, especially in motion.

## Operating characteristics
- **Time Complexity:** $O(N^2 \log N)$ per view when implemented via the Fourier Projection-Slice Theorem, avoiding the $O(N^3)$ cost of digital compositing [G].
- **Memory Complexity:** Not given in the sources [G].
- **Strengths:** Efficient rendering of depth-cued X-ray images; leverages fast Fourier transforms [S1, unofficial].
- **Limitations:** Approximates exponential attenuation; does not model occlusion; accuracy depends on the linearity assumption of the depth cue [S1, unofficial].
- **When to use:** Not given in the sources [G].
- **When not to use:** Not given in the sources [G].

## Background
The method was introduced by Marc Levoy in his work on volume rendering using the Fourier Projection-Slice Theorem [S1, unofficial]. It addresses the computational cost of exponential attenuation by linearizing the absorption term, allowing the use of frequency domain methods for shading [S1, unofficial]. The approach is based on the observation that linear attenuation with a constant coefficient can be expressed as a Fourier projection, enabling efficient rendering [S1, unofficial].

## Sources
[S1] PDF Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S2] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
