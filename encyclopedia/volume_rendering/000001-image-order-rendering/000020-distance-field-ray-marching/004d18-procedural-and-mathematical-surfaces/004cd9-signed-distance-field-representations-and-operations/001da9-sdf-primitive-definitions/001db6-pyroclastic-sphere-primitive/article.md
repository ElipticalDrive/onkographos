# Pyroclastic Sphere Primitive

## Overview
The Pyroclastic Sphere Primitive displaces a base sphere's signed distance field with procedural noise to generate organic volumetric shapes with overhangs [S1, unofficial]. It distinguishes itself from standard SDF primitives by applying noise directly to the distance function and supports 2D or 3D displacement modes to control surface topology and antialiasing via smoothstep [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** World position $\mathbf{p}$, sphere center $\mathbf{c}$, radius $r$, noise function $m(\cdot)$, amplitude $A$, gamma correction $\gamma$, 2D displacement flag $d_{2D}$, voxel size $\mathbf{v}$, base density $\rho_0$ [S1, unofficial].
*   **Output:** Sampled density value $\rho(\mathbf{p})$ [S1, unofficial].

**Data Structures:**
The primitive stores parameters as scalar or vector fields: `radius` ($r$), `wsCenter` ($\mathbf{c}$), `amplitude` ($A$), `gamma` ($\gamma$), `displace2D` ($d_{2D}$), `density` ($\rho_0$), and a reference to a procedural noise evaluator (typically fractal Brownian motion) [S1, unofficial].

**Steps:**
1.  **Transform to Local Space:** Compute the local position relative to the sphere center, normalized by the radius: $\mathbf{lsP} = (\mathbf{p} - \mathbf{c}) / r$ [S1, unofficial].
2.  **Determine Noise Sample Point:** Set the noise evaluation point $\mathbf{nsP}$: if $d_{2D}$ is true, project $\mathbf{lsP}$ onto the unit sphere surface ($\mathbf{nsP} = \mathbf{lsP} / \|\mathbf{lsP}\|$); otherwise, use $\mathbf{lsP}$ directly [S1, unofficial].
3.  **Evaluate Noise:** Evaluate the procedural noise function at $\mathbf{nsP}$, apply gamma correction, and scale by amplitude: $\mathbf{fractalFunc} = A \cdot \gamma(m(\mathbf{nsP}))$ [S1, unofficial].
4.  **Compute Sphere Distance:** Calculate the signed distance from the sphere surface in local space: $d_{sphere} = \|\mathbf{lsP}\| - 1.0$ [S1, unofficial].
5.  **Compute Filter Width:** Determine the antialiasing threshold width from the world-space voxel size: $w_{filter} = \|\mathbf{v}\|$ and $w_{threshold} = w_{filter} \cdot 0.5 / r$ [S1, unofficial].
6.  **Compute Pyroclastic Value:** Combine the sphere distance and noise displacement using a linear clamp interpolation (`fit`) centered at $0$ with width $w_{threshold}$: $v_{pyro} = \text{fit}(d_{sphere} - \mathbf{fractalFunc}, -w_{threshold}, w_{threshold}, 1.0, 0.0)$ [S1, unofficial].
7.  **Output Density:** Scale the pyroclastic value by the base density: $\rho(\mathbf{p}) = \rho_0 \cdot v_{pyro}$ [S1, unofficial].

**Formulas:**
The source text provides an alternative analytical formula: $pyroclasticDensity(\mathbf{P}) = \max(r - \|\mathbf{P}/r\| + |m(\mathbf{P})|, 0)$ [S1, unofficial]. However, the provided implementation code uses the `fit`-based smoothstep blending described in Step 6 [S1, unofficial]. Both approaches are cited from the same source [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pyroclastic_sphere_sample(p_world, center, radius, noise_func, amplitude, gamma, displace_2d, voxel_size, base_density):
    # 1. Transform to local space
    local_p = (p_world - center) / radius

    # 2. Determine noise sample point
    if displace_2d:
        norm_local = length(local_p)
        if norm_local < 1e-6:
            noise_p = local_p # [G] Fallback for origin to avoid division by zero
        else:
            noise_p = local_p / norm_local
    else:
        noise_p = local_p

    # 3. Evaluate noise with gamma and amplitude
    raw_noise = noise_func(noise_p)
    gamma_noise = raw_noise ** gamma # [G] Standard gamma correction exponentiation
    fractal_func = amplitude * gamma_noise

    # 4. Compute sphere distance
    sphere_func = length(local_p) - 1.0

    # 5. Compute filter width
    filter_width = length(voxel_size)
    threshold_width = filter_width * 0.5 / radius

    # 6. Compute pyroclastic value using fit (linear clamp interpolation)
    pyro_value = fit(sphere_func - fractal_func, -threshold_width, threshold_width, 1.0, 0.0)

    # 7. Output density
    return base_density * pyro_value

def length(v):
    return (v.x**2 + v.y**2 + v.z**2)**0.5

def fit(x, a, b, c, d):
    t = max(0.0, min(1.0, (x - a) / (b - a)))
    return c + t * (d - c)
```

**Parameters:**
*   `radius`: Radius of the base sphere [S1, unofficial].
*   `amplitude`: Scaling factor for the noise displacement [S1, unofficial].
*   `gamma`: Exponent for gamma correction applied to the noise function output [S1, unofficial].
*   `displace2D`: Boolean flag controlling displacement mode [S1, unofficial].
*   `density`: Base density multiplier [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overhangs:** If `displace2D` is false, high noise amplitude can cause the density to disconnect from the main body, creating floating artifacts [S1, unofficial].
*   **Origin Singularity:** When $\mathbf{lsP}$ is near zero, normalization for 2D displacement is unstable [G].
*   **Precision:** The subtraction $d_{sphere} - \mathbf{fractalFunc}$ can suffer from cancellation errors if the noise amplitude is large relative to the sphere distance [G].

## Subtopics

### [Deforming SDF Sphere with Noise](1759-deforming-sdf-sphere-with-noise.md)
This method defines an implicit surface by displacing the radius of a base sphere using animated procedural noise, creating a time-varying organic shape. It distinguishes itself from static SDF primitives by combining a low-frequency sinusoidal "breathing" modulation with high-frequency fractal Brownian motion (fBM) and sinusoidal products to generate complex surface topology.

## Operating characteristics
Not given in the sources.

## Background
The Pyroclastic Sphere Primitive was introduced in the context of production volume rendering, specifically within the framework of solid noise primitives [S1, unofficial]. The technique leverages the concept of using noise as a displacement field on a distance function, a common practice in procedural modeling and texturing [S1, unofficial]. The specific "pyroclastic" naming and the implementation details regarding 2D versus 3D displacement for controlling overhangs are detailed in the source material [S1, unofficial].

## Sources
[S1] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
