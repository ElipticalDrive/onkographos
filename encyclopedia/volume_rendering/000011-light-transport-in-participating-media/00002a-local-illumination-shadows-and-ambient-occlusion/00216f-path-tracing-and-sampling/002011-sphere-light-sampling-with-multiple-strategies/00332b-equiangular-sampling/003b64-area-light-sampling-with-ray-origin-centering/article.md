# Area Light Sampling with Ray-Origin Centering

Also called: Ray-Origin Centering for Area Lights

## Overview
This method generalizes [EquiAngular Sampling](2272-equiangular-sampling.md) to area lights of arbitrary shapes by using the sampled light point as the center for the ray-segment sampling distribution. It solves the high variance at light edges caused by approximating the distance $D$ with a single arbitrary point (like the light center), which fails for large, stretched, or textured lights. Unlike approximations where the $D^2$ term in the PDF dominates and causes noisy edges, this method distributes the error uniformly over the light's surface.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Area light geometry (surface definition).
- Phase function $p(\mathbf{d}, \mathbf{d}')$.

**Outputs:**
- A sampled point $\mathbf{l}$ on the light surface.
- A sampled point $\mathbf{p}$ on the ray segment between $\mathbf{o}$ and $\mathbf{l}$.
- The probability density function (PDF) values for MIS.

**Data Structures:**
- Light surface representation capable of evaluating position and PDF for uniform sampling.
- Ray structure with origin $\mathbf{o}$ and direction $\mathbf{d}$.

**Steps:**
1. **Sample Light Point:** Sample a point $\mathbf{l}$ uniformly on the area light surface. Let $p_{light}(\mathbf{l})$ be the PDF of this sample.
2. **Calculate Distance:** Compute the distance $D$ from the sampled light point $\mathbf{l}$ to the ray origin $\mathbf{o}$. Note that $D$ is now a function of the position sampled from the light: $D(\mathbf{l}) = \|\mathbf{l} - \mathbf{o}\|$.
3. **Determine Sampling Distribution:** Use the sampled light point $\mathbf{l}$ as the center for the equi-angular sampling equations. Specifically, use $\mathbf{l}$ in place of the fixed center used in standard equi-angular sampling to determine the sampling distribution along the ray segment between $\mathbf{o}$ and $\mathbf{l}$. This involves calculating the angle $\theta$ between the ray direction $\mathbf{d}$ and the vector $\mathbf{l} - \mathbf{o}$, and sampling the distance $t$ along the ray based on the equiangular distribution centered at $\mathbf{l}$.
4. **Sample Ray Point:** Sample a point $\mathbf{p}$ on the ray segment using the distribution derived in Step 3. Let $p_{ray}(\mathbf{p} | \mathbf{l})$ be the conditional PDF of this sample given $\mathbf{l}$.
5. **Apply MIS:** Apply Multiple Importance Sampling (MIS) between the area light surface sampling and the phase function sampling to mask remaining noise. The combined weight is typically calculated using the balance heuristic:
   $$ w = \frac{p_{light}(\mathbf{l}) p_{ray}(\mathbf{p} | \mathbf{l})}{p_{light}(\mathbf{l}) p_{ray}(\mathbf{p} | \mathbf{l}) + p_{phase}(\mathbf{p})} $$
   where $p_{phase}$ is the PDF of sampling the ray point based on the phase function alone.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_area_light_with_ray_origin_centering(ray_origin, ray_dir, light):
    # Step 1: Sample a point on the area light surface
    l, pdf_light = light.sample_uniform()
    
    # Step 2: Calculate distance from sampled light point to ray origin
    # D is a function of the position sampled from the light
    vec_to_light = l - ray_origin
    D = length(vec_to_light)
    
    # Step 3: Use the sampled light point as the center for equi-angular sampling
    # Calculate angle theta between ray direction and vector to light
    cos_theta = dot(normalize(ray_dir), normalize(vec_to_light))
    
    # Sample distance t along the ray using equi-angular distribution centered at l
    # This uses the standard equi-angular sampling equations but with l as the center
    # The specific formula depends on the implementation of equi-angular sampling
    # typically involving sampling u in [0,1] and computing t based on theta and D
    u = random_float(0.0, 1.0)
    t = sample_equiangular_distance(u, cos_theta, D)
    
    # Step 4: Compute the sampled point on the ray
    p = ray_origin + t * ray_dir
    
    # Step 5: Calculate PDF for ray sampling given the light point
    # This is the conditional PDF p_ray(p | l)
    pdf_ray = compute_equiangular_pdf(t, cos_theta, D)
    
    # Step 6: Apply MIS with phase function
    # Calculate phase function PDF
    phase_pdf = light.phase_function.pdf(ray_dir, normalize(l - p))
    
    # Balance heuristic weight
    numerator = pdf_light * pdf_ray
    denominator = numerator + pdf_light * phase_pdf # Note: pdf_light is common factor if considering joint PDF
    # Actually, MIS weight for light sampling vs phase sampling:
    # w = (p_light(p) * p_ray(p|l)) / (p_light(p) * p_ray(p|l) + p_phase(p) * p_light(p))
    # Simplifies to: w = pdf_ray / (pdf_ray + phase_pdf) if we consider the light sampling part as fixed
    # However, standard MIS for light vs phase usually weights the light sample PDF against the phase sample PDF
    # The exact formulation depends on the specific MIS strategy (balance, power, etc.)
    # Using balance heuristic:
    weight = numerator / (numerator + pdf_light * phase_pdf)
    
    return p, weight, pdf_light, pdf_ray
```

**Edge Cases and Pitfalls:**
- Choosing a sampling center too close to the ray can cause a potential singularity. This method does not solve this singularity but distributes the error uniformly over the light's surface [S1].
- For very large lights, the approximation of using the sampled point as the center is more accurate than using the light center, but precision issues may arise if $D$ is very small.

## Subtopics

### Spherical Rectangle Solid Angle Sampling
A sampling technique for area lights that treats them as spherical rectangles to improve PDF accuracy at small solid angles [S2, unofficial]. It uses a threshold of $10^{-3}$ steradians for switching to spherical rectangle sampling [S2, unofficial]. This improves PDF for rectangular lights at small solid angles compared to standard planar sampling [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, assuming constant-time light sampling and distance calculation.
- **Memory Complexity:** $O(1)$ additional memory.
- **Performance:** Substantially reduces variance compared to approximations using the light center, especially for large, stretched, or textured area lights [S1].
- **Strengths:** Works for large, stretched, or textured area lights where arbitrary center approximations fail [S1]. Distributes error uniformly over the light's surface [S1].
- **Limitations:** Does not solve the singularity issue when the sampling center is too close to the ray [S1]. Requires accurate light sampling and distance calculation.
- **When to Use:** When rendering scenes with large, stretched, or textured area lights in participating media.
- **When Not to Use:** When light sources are very small and far away, where standard approximations may suffice, or when computational resources are extremely limited and variance reduction is not critical.

## Background
This method was introduced as a generalization of equi-angular sampling for area lights in participating media [S1]. It addresses the limitations of previous approximations that used arbitrary points like the light center, which failed for large lights and caused noisy edges due to the $D^2$ term dominating the PDF [S1]. The technique is applicable to path tracing and bidirectional path tracing [S1].

## Sources
[S1] Importance sampling of area lights in participating media, https://doi.org/10.1145/2037826.2037899, fetched 2026-11-02
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
