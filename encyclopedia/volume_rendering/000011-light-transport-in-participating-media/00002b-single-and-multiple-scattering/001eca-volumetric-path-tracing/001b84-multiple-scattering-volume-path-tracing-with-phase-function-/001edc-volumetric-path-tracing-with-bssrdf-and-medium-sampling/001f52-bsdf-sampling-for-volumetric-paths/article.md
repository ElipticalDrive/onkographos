# BSDF Sampling for Volumetric Paths

Also called: Implicit Path Tracing with Volume Interaction, Surface scattering in volumetric path tracing

## Overview
This method handles the surface interaction component of volumetric path tracing, calculating indirect lighting contributions based on surface BRDF/BSDF properties and sampling strategies [S1]. It determines the new volumetric path direction upon surface intersection, updating throughput and rescaled probabilities to maintain unbiased estimation [S2]. It differs from pure volume scattering integrators by explicitly managing the coupling between surface geometry and participating media, specifically handling the transition from a surface hit back into the medium or to another surface [S2].

## How it works
**Inputs:**
- Intersection data (`isect`): Contains surface normal (`isect.shading.n`), incoming ray direction (`isect.wo`), and optional BSSRDF data [S2].
- BSDF object (`bsdf`): Defines the surface reflectance properties [S2].
- Sampler (`sampler`): Provides random numbers (`Get1D()`, `Get2D())` [S2].
- Current throughput (`beta`): Accumulated path weight [S2].
- Rescaled probabilities (`r_u`, `r_l`): For Russian Roulette or importance rescaling [S2].
- Sample mode (`sampleMode`): Defines the PDF calculation strategy (e.g., Uniform Hemisphere, Cosine Weighted) [S1].
- Scene objects and lights: For recursive calls or light sampling [S1].

**Outputs:**
- Updated throughput (`beta`) [S2].
- Rescaled probabilities (`r_l`) [S2].
- New ray (`ray`) for continuation [S2].
- Indirect radiance contribution (`L_indirect`) [S1].
- State flags (`specularBounce`, `anyNonSpecularBounces`, `etaScale`) [S2].

**Data Structures:**
- `BSDFSample`: Contains `f` (BSDF value), `wi` (outgoing direction), `pdf` (probability density), `pdfIsProportional` (boolean flag), `IsSpecular()` (boolean), `IsTransmission()` (boolean), `eta` (refraction index), `flags` [S2].
- `Interaction`/`Intersection`: Contains `wo` (incoming direction), `shading.n` (normal), `SpawnRay()` method [S2].

**Steps:**

1. **Calculate Cosine Factor:**
   Compute the cosine of the angle between the surface normal and the ray direction:
   $$ \cos\theta = \text{clamp}(\mathbf{n} \cdot \mathbf{w}) $$
   where $\mathbf{n}$ is the surface normal and $\mathbf{w}$ is the ray direction [S1].

2. **Sample BSDF:**
   Generate random numbers and sample the BSDF to determine the outgoing direction $\mathbf{w}_o$ (stored as `bs.wi`):
   ```python
   wo = isect.wo
   u = sampler.Get1D()
   bs = bsdf.Sample_f(wo, u, sampler.Get2D())
   if bs is None:
       break
   ```
   [S2]

3. **Update Throughput:**
   Multiply the current throughput `beta` by the BSDF value, the cosine factor, and divide by the PDF:
   $$ \beta_{\text{new}} = \beta_{\text{old}} \times \frac{f(\mathbf{w}_i, \mathbf{w}_o) \cdot |\mathbf{n} \cdot \mathbf{w}_o|}{\text{pdf}} $$
   In code:
   ```python
   beta *= bs.f * abs(bs.wi.dot(isect.shading.n)) / bs.pdf
   ```
   [S2]

4. **Update Rescaled Probabilities:**
   Adjust the rescaled path probabilities based on whether the BSDF PDF is proportional to the actual PDF:
   ```python
   if bs.pdfIsProportional:
       r_l = r_u / bsdf.pdf(wo, bs.wi)
   else:
       r_l = r_u / bs.pdf
   ```
   [S2]

5. **Update Path State:**
   - Determine if the bounce is specular:
     ```python
     specularBounce = bs.is_specular()
     anyNonSpecularBounces = anyNonSpecularBounces or not bs.is_specular()
     ```
   - If transmission, update the refraction index scale:
     ```python
     if bs.is_transmission():
         etaScale *= bs.eta ** 2
     ```
   - Spawn the new ray into the volume or scene:
     ```python
     ray = isect.spawn_ray(ray, bsdf, bs.wi, bs.flags, bs.eta)
     ```
   [S2]

6. **Handle BSSRDF (Optional/Conditional):**
   If the surface supports subsurface scattering and the interaction is transmission, sample the BSSRDF:
   ```python
   bssrdf = isect.get_bssrdf(ray, lambda, camera, scratch_buffer)
   if bssrdf and bs.is_transmission():
       uc = sampler.Get1D()
       up = sampler.Get2D()
       probe_seg = bssrdf.sample_sp(uc, up)
       if probe_seg is None:
           break
       seed = mix_bits(float_to_bits(sampler.Get1D()))
       # [G] Standard reservoir sampling initialization for subsurface interactions
       interaction_sampler = WeightedReservoirSampler(seed)
       # [G] Intersect BSSRDF sampling ray against scene geometry
   ```
   [S2]

7. **Calculate Indirect Radiance:**
   Recursively trace the new ray to compute indirect lighting:
   ```python
   L_indirect = hit_obj.get_object_color() * get_pixel_color_implicit_path_tracing(objects, index_of_closest_object, hit_t, second_ray, light, depth_i + 1, sample_mode) * beta
   ```
   [S1]

8. **Accumulate Result:**
   Add the indirect radiance to the pixel color:
   $$ C = C + L_{\text{indirect}} $$
   [S1]

**Parameters:**
- `brdf`: Bidirectional Reflectance Distribution Function value. In some implementations, this is hardcoded as $1/\pi$ [S1].
- `sampleMode`: Sampling strategy (e.g., `Uniform_Hemisphere_Solid_Angle`, `Cosine_Weighted`). Affects PDF calculation [S1].
  - If `sampleMode` is `Uniform_Hemisphere_Solid_Angle`, set `pdf = 1 / (2 * \pi)` and update `throughput *= brdf * cosine / pdf` [S1].
  - If `sampleMode` is `Cosine_Weighted`, set `pdf = 1 / \pi` and update `throughput *= brdf / pdf` [S1].
  - If `sampleMode` is neither, raise an error [S1].

**Edge Cases and Pitfalls:**
- **No BSDF Sample:** If `bsdf.Sample_f` returns null, the path terminates [S2].
- **PDF Proportionality:** The formula for `r_l` changes based on `pdfIsProportional` [S2].
- **Transmission Refraction:** The `etaScale` must be updated correctly for multiple refractions [S2].
- **Undefined Sample Mode:** If `sampleMode` is neither Uniform Hemisphere nor Cosine Weighted, an error should be raised [S1].

## Operating characteristics
- **Time Complexity:** Depends on the BSDF complexity and scene depth [S2].
- **Strengths:** Handles complex surface interactions within volumetric paths, including subsurface scattering [S2].
- **When to Use:** Scenes with complex surface materials and participating media [S2].

## Background
This method is part of the broader field of volumetric path tracing, specifically addressing the coupling between surface and volume interactions [S2]. It builds upon the principles of path tracing [S2] and incorporates techniques for handling BSSRDFs in participating media [S2]. The specific implementation details in [S1] reflect an early or simplified approach to implicit path tracing with volume interactions [S1].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
