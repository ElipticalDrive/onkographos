# Next Event Estimation for Volumetric Path Tracing

Also called: Direct Illumination Sampling, Direct lighting estimation, Light sampling, SampleLd, Vol Path Tracing 3, Volumetric NEE, Volumetric Path Tracing with Next Event Estimation

## Overview
Next Event Estimation (NEE) for volumetric path tracing estimates direct illumination in participating media by explicitly sampling points on light sources and tracing shadow rays through volumes and index-matched surfaces [S1]. Unlike phase function sampling, which is inefficient for small light sources, NEE changes the integration domain from the hemisphere to the light surface area, accounting for transmittance via ratio-tracking and handling index-matched boundaries without terminating the shadow ray [S1]. It builds on [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by adding direct lighting estimation to the path vertices [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Current scattering position $\mathbf{p}$ and view direction $\omega$ (or surface normal if at a boundary) [S3].
*   Current medium properties $\lambda$ (spectral density) and phase function $p(\omega, \omega')$ [S3].
*   Path throughput $\beta$ accumulated from the camera to $\mathbf{p}$ [S3].
*   Sampler state for random number generation [S3].
*   Scene geometry and light sources [S3].

**Outputs:**
*   Estimated direct illumination radiance contribution $L_{dir}(\mathbf{p}, \omega)$ [S3].

### Data Structures
*   **LightSampleContext**: Holds the interaction point and flags for surface offsetting [S3].
*   **SampledLight**: Contains the pointer to the light source and its selection probability $p$ [S3].
*   **LightLiSample**: Contains the sampled point on the light $\mathbf{p}_{light}$, incoming direction $\omega_i$, light radiance $L$, PDF $pdf$, and light type [S3].
*   **MediumProperties**: Contains absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and extinction coefficient $\sigma_t = \sigma_a + \sigma_s$ [S3].
*   **Shadow Ray State**: Variables for ratio-tracking transmittance estimation: $T_{ray}$ (current transmittance estimate), $r_l$ (lower bound PDF weight), $r_u$ (upper bound PDF weight) [S3].

### Numbered Steps

1.  **Context Initialization**: If the current vertex is a surface interaction with a BSDF, initialize `LightSampleContext` with the surface interaction [S3]. If the BSDF is reflective but not transmissive, offset the sampling origin in the direction of the outgoing vector $\omega_{out}$: $\mathbf{p}_{sample} = \mathbf{p} + \epsilon \omega_{out}$ [S3]. If the BSDF is transmissive but not reflective, offset in the opposite direction: $\mathbf{p}_{sample} = \mathbf{p} - \epsilon \omega_{out}$ [S3]. If the vertex is in a medium (no BSDF), use the medium interaction point directly [S3].

2.  **Light Source Sampling**: Sample a light source index $u_1$ from the sampler [S3]. Select a light source $L_{sel}$ using the light sampler's `Sample` method [S3]. If no light is selected, return 0 [S3]. Sample a point on the light surface using $u_2$ (2D uniform) [S3]. Get the light sample structure `ls` containing $\mathbf{p}_{light}$, $\omega_i$, $L_e$, and $pdf_{light}$ [S3]. Compute the total PDF for the light selection: $pdf_{dir} = p(L_{sel}) \cdot pdf_{light}$ [S3]. If $pdf_{dir} == 0$, return 0 [S3].

3.  **Scattering Function Evaluation**: Define the outgoing direction from the vertex to the light: $\omega_i = \text{normalize}(\mathbf{p}_{light} - \mathbf{p})$ [S3]. If a BSDF exists: evaluate the BSDF: $f_r = f_{bsdf}(\omega_{out}, \omega_i)$ [S3]. Compute the geometry term: $G = |\omega_i \cdot \mathbf{n}|$ [S3]. $f_{hat} = f_r \cdot G$ [S3]. $pdf_{phase} = f_{bsdf}.PDF(\omega_{out}, \omega_i)$ [S3]. If in a medium (Phase Function): evaluate the phase function: $f_{hat} = p(\omega_{out}, \omega_i)$ [S3]. $pdf_{phase} = p.PDF(\omega_{out}, \omega_i)$ [S3]. If $f_{hat} == 0$, return 0 [S3].

4.  **Shadow Ray Tracing and Transmittance Estimation**: Initialize the shadow ray from $\mathbf{p}$ to $\mathbf{p}_{light}$ [S3]. Initialize ratio-tracking variables: $T_{ray} = 1.0$, $r_l = 1.0$, $r_u = 1.0$ [S3]. Initialize a random number generator for Russian Roulette [S3]. Loop while the ray direction is non-zero [S3]:
    1.  **Intersection**: Trace the shadow ray against the scene [S3]. Let $t_{hit}$ be the distance to the closest intersection, or $\infty$ if none [S3].
    2.  **Opaque Occlusion**: If an intersection occurs with an opaque material (material ID $\ge 0$), the light is blocked [S1][S3]. Return 0 [S3].
    3.  **Index-Matched Surfaces**: If the intersection is with an index-matched surface (material ID $< 0$, no material): increment a shadow bounce counter [S1]. Check max depth limits (path bounces + shadow bounces) [S1]. If exceeded, return 0 [S1]. Update the current medium to the medium on the other side of the surface [S1]. Continue the ray through the surface (spawn new ray segment to $\mathbf{p}_{light}$) [S1].
    4.  **Transmittance Update (Ratio-Tracking)**: Determine the segment length $t_{seg} = \min(t_{hit}, t_{max})$ [S3]. If the segment is within a medium: sample a distance $t_{sample}$ using inverse transform sampling for the exponential distribution: $t_{sample} = \frac{\ln(1-u)}{-\sigma_t}$ [S1]. Calculate the majorant extinction $\sigma_{maj}$ (typically $\sigma_t$ or a precomputed upper bound) [S3]. Calculate the non-absorbing part: $\sigma_n = \max(0, \sigma_{maj} - \sigma_a - \sigma_s)$ [S3]. Update the transmittance estimate and PDF weights: $T_{ray} \leftarrow T_{ray} \cdot \frac{T_{maj}(t_{seg}) \cdot \sigma_n}{T_{maj}(0) \cdot \sigma_{maj}}$, $r_l \leftarrow r_l \cdot \frac{T_{maj}(t_{seg}) \cdot \sigma_{maj}}{T_{maj}(0) \cdot \sigma_{maj}}$, $r_u \leftarrow r_u \cdot \frac{T_{maj}(t_{seg}) \cdot \sigma_n}{T_{maj}(0) \cdot \sigma_{maj}}$ [S3]. *(Note: $T_{maj}(t) = e^{-\sigma_{maj} t}$)* [S3]. **Russian Roulette**: If the estimated transmittance $T_{ray} / (r_l + r_u)/2$ is very small (e.g., max component $< 0.05$), terminate with probability $q=0.75$ or scale up $T_{ray}$ by $1/(1-q)$ [S3].
    5.  **Final Segment**: If no intersection ($t_{hit} = \infty$), break the loop (light is visible) [S3]. If intersection occurred (index-matched), update the ray origin and direction to the new segment towards $\mathbf{p}_{light}$ [S3].

5.  **Contribution Calculation and MIS**: If $T_{ray} == 0$, return 0 [S3]. Compute the geometry term for the change of variable from solid angle to area: $G = \frac{|\omega_i \cdot \mathbf{n}_{light}|}{\|\mathbf{p}_{light} - \mathbf{p}\|^2}$ [S1]. The NEE PDF (area measure) is $pdf_{nee} = pdf_{dir}$ [S1]. The Phase Function PDF (solid angle measure) converted to area measure is $pdf_{phase\_area} = pdf_{phase} \cdot G$ [S1]. Compute MIS weights using the power heuristic: $w = \frac{pdf_{nee}^2}{pdf_{nee}^2 + pdf_{phase\_area}^2}$ [S1]. Calculate the direct lighting contribution: $L_{dir} = \beta \cdot f_{hat} \cdot T_{ray} \cdot L_e(\mathbf{p}_{light}) \cdot G$ [S1]. Apply MIS weight: $L_{dir} = w \cdot L_{dir}$ [S1]. Return $L_{dir}$ [S1][S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from math import log, exp, sqrt, abs

class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __sub__(self, v): return Vec3(self.x-v.x, self.y-v.y, self.z-v.z)
    def __mul__(self, s): return Vec3(self.x*s, self.y*s, self.z*s)
    def length(self): return sqrt(self.x**2 + self.y**2 + self.z**2)
    def normalize(self): l = self.length(); return Vec3(self.x/l, self.y/l, self.z/l) if l > 1e-12 else Vec3(0,0,0)
    def dot(self, v): return self.x*v.x + self.y*v.y + self.z*v.z
    def abs_dot(self, v): return abs(self.dot(v))

class Ray:
    def __init__(self, origin, direction): self.o = origin; self.d = direction

class RNG:
    def __init__(self, seed1, seed2): self.state = (seed1 + seed2) % (2**32)
    def uniform(self): self.state = (self.state * 1103515245 + 12345) & 0x7fffffff; return self.state / 0x7fffffff

class LightSampleContext:
    def __init__(self, point, offset_dir=None): self.pi = point; self.offset_dir = offset_dir

class SampledLight:
    def __init__(self, light, p): self.light = light; self.p = p

class LightLiSample:
    def __init__(self, p_light, wi, L, pdf, is_delta): self.p_light = p_light; self.wi = wi; self.L = L; self.pdf = pdf; self.is_delta = is_delta

class MediumProperties:
    def __init__(self, sigma_a, sigma_s): self.sigma_a = sigma_a; self.sigma_s = sigma_s; self.sigma_t = sigma_a + sigma_s

def next_event_estimation(p, current_medium, phase_function, beta, sampler, scene, lights, r_p):
    # 1. Context Initialization
    ctx = LightSampleContext(p)
    
    # 2. Light Source Sampling
    u1 = sampler.uniform()
    sampled_light = lights.sample(ctx, u1)
    if sampled_light is None: return 0.0
    
    u2 = sampler.uniform(), sampler.uniform()
    ls = sampled_light.light.sample_li(ctx, u2, sampler.spectral_sample(), True)
    if ls is None or ls.pdf == 0 or ls.L == 0: return 0.0
    
    pdf_dir = sampled_light.p * ls.pdf
    
    # 3. Scattering Function Evaluation
    wo = -Vec3(0, 0, 1).normalize() # View direction placeholder
    wi = (ls.p_light - p).normalize()
    
    if phase_function:
        f_hat = phase_function.p(wo, wi)
        scatterPDF = phase_function.pdf(wo, wi)
    else:
        f_hat = 0.0
        scatterPDF = 0.0
        
    if f_hat == 0: return 0.0
    
    # 4. Shadow Ray Tracing and Transmittance Estimation
    light_ray = Ray(p, ls.p_light - p)
    T_ray = 1.0
    r_l = 1.0
    r_u = 1.0
    rng = RNG(hash(light_ray.o.x), hash(light_ray.o.y))
    shadow_bounces = 0
    max_shadow_bounces = 10
    
    while light_ray.d.length() > 1e-6:
        isect = scene.intersect(light_ray, 1 - 1e-4)
        t_hit = isect.t if isect else float('inf')
        
        if isect and isect.material_id >= 0:
            return 0.0
            
        t_max = t_hit if isect else 1 - 1e-4
        
        if current_medium:
            sigma_maj = current_medium.sigma_t
            u = rng.uniform()
            t_sample = log(1 - u) / -current_medium.sigma_t
            t_sample = min(t_sample, t_max)
            
            T_maj = exp(-sigma_maj * t_sample)
            pdf = 1.0 * sigma_maj
            sigma_n = max(0, sigma_maj - current_medium.sigma_a - current_medium.sigma_s)
            
            T_ray *= T_maj * sigma_n / pdf
            r_l *= T_maj * sigma_maj / pdf
            r_u *= T_maj * sigma_n / pdf
            
            avg_r = (r_l + r_u) / 2.0
            if avg_r > 0:
                Tr = T_ray / avg_r
                if Tr < 0.05:
                    q = 0.75
                    if rng.uniform() < q:
                        T_ray = 0.0
                    else:
                        T_ray /= (1 - q)
            
            T_ray *= T_maj / 1.0
            r_l *= T_maj / 1.0
            r_u *= T_maj / 1.0
            
        if T_ray == 0: return 0.0
        if not isect: break
        
        shadow_bounces += 1
        if shadow_bounces > max_shadow_bounces: return 0.0
        
        current_medium = scene.update_medium(isect, current_medium)
        light_ray = Ray(isect.position, ls.p_light - isect.position)
        
    # 5. Contribution Calculation and MIS
    if T_ray == 0: return 0.0
    
    dist_sq = (ls.p_light - p).length() ** 2
    G = wi.abs_dot(Vec3(0, 0, 1)) / dist_sq # Placeholder normal for light
    
    pdf_phase_area = scatterPDF * G
    pdf_nee = pdf_dir
    
    w = (pdf_nee ** 2) / (pdf_nee ** 2 + pdf_phase_area ** 2)
    
    L_dir = beta * f_hat * T_ray * ls.L * G
    return w * L_dir
```

### Parameters and Edge Cases
*   **Shadow Epsilon**: A small offset (e.g., $10^{-4}$) to prevent self-intersection when spawning shadow rays [S3].
*   **Max Shadow Bounces**: Limits the recursion depth for index-matched surfaces to prevent infinite loops in complex glass volumes [S1].
*   **Ratio-Tracking Majorant**: $\sigma_{maj}$ should be a tight upper bound on $\sigma_t$ to minimize Russian Roulette frequency. If $\sigma_{maj}$ is too loose, variance increases.
*   **Delta Lights**: If the light is a delta emitter (e.g., point light), the MIS weight simplifies to 1 for the NEE path, as phase function sampling cannot hit a delta light directly [S3].

## Operating characteristics

*   **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of light sources and $M$ is the average number of intersections/transmittance samples per shadow ray. Ratio-tracking adds overhead proportional to the optical thickness of the medium [S1][S3].
*   **Memory Complexity**: $O(1)$ additional memory per shadow ray trace, excluding scene acceleration structures [S3].
*   **Performance**: Significantly reduces variance for scenes with small, bright light sources compared to pure phase function sampling. Performance degrades in highly participating media with high optical thickness due to the cost of ratio-tracking and Russian Roulette [S1][S3].
*   **Strengths**: Unbiased estimation of direct lighting; handles index-matched surfaces correctly; reduces noise for small lights [S1].
*   **Limitations**: Requires efficient shadow ray tracing; ratio-tracking can be computationally expensive in dense media; MIS weight calculation requires accurate PDFs for both methods [S1][S3].

## Background

Next Event Estimation for volumetric path tracing was introduced as an extension to volumetric path tracing algorithms to improve the efficiency of direct lighting estimation [S1]. The method changes the integration domain from the hemisphere to the light surface area, requiring careful handling of transmittance through participating media and index-matched boundaries [S1].

## Sources

[S1] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
