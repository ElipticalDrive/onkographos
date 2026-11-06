# Shadow Volume Rendering

Also called: Volumetric Shadow Maps

## Overview
Shadow Volume Rendering is a specialized ray marching routine that computes only the accumulated alpha (transmittance) along a ray to generate shadow maps or occlusion values for volume scenes [S1, unofficial]. It solves the problem of self-shadowing in participating media by precomputing a shadow volume that stores the amount of light arriving at each sample after attenuation by intervening material [S2]. Unlike full-resolution ray integration, it decouples storage from integration, allowing interactive adjustment of shadow map density while maintaining high-quality visibility estimates [S1, unofficial]. It differs from [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by explicitly precomputing a shadow volume structure rather than relying solely on runtime opacity shadow maps, and from [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md) by focusing on alpha accumulation for shadow generation rather than light strength buffering.

## How it works
**Inputs:**
- Ray origin and direction in world space.
- World-to-object transformation matrix (`xfmPoint`, `xfmVector`).
- Ray bounds `tMin`, `tMax`.
- Sampling scale multiplier (`samplingScale`).
- Step size (`self.step`).
- Random number generator state (`RandomTEA`).

**Outputs:**
- Accumulated alpha (transmittance) value.
- Volumetric shadow map data (precomputed).

**Data Structures:**
- `ShadowPayload`:
  - `rng`: Pointer to `RandomTEA` instance.
  - `alpha`: Float, accumulated alpha.
  - Layout: Packaged via `packPointer(&payload, u0, u1)`.

**Steps:**
1. **Transform Ray:** Convert ray origin and direction from world space to object space.
   ```python
   org = xfmPoint(wto, _org)
   dir = xfmVector(wto, _dir)
   ```
2. **Initialize Payload:** Create `ShadowPayload` with random state and initial alpha of 0.
3. **March Ray:** Iterate along the ray using `raymarching_iterator` with parameters `org`, `dir`, `tMin`, `tMax`, `samplingScale * self.step`, and a lambda function.
4. **Sample Position:** Compute sample position with jitter.
   ```python
   p = org + lerp(jitter, t.x, t.y) * dir
   ```
5. **Accumulate Alpha:** Perform front-to-back blending.
   ```python
   payload.alpha += (1.0 - payload.alpha) * sampleAlpha
   ```
6. **Termination:** Continue marching while `payload.alpha < nearly_one`.

**Parameters:**
- `samplingScale`: Multiplier applied to `self.step` to adjust shadow ray sampling density. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Blurry Shadows:** Results in visual artifacts such as blurry shadows [S2].
- **Dark Images:** Can result in dark images due to over-attenuation [S2].
- **Memory Usage:** Uses extra memory for storing the shadow volume [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shadow_volume_rendering(ray_org, ray_dir, wto, t_min, t_max, sampling_scale, step_size, random_tea):
    # Transform ray to object space
    org = xfmPoint(wto, ray_org)
    dir = xfmVector(wto, ray_dir)
    
    # Initialize payload
    payload = ShadowPayload()
    payload.rng = random_tea
    payload.alpha = 0.0
    
    # March along the ray
    # Note: raymarching_iterator is assumed to handle the loop and lambda invocation
    # The lambda function performs sampling and accumulation
    def sample_lambda(t):
        # Compute sample position with jitter
        jitter = get_jitter(payload.rng) # [G] Assuming a jitter generation function
        p = org + lerp(jitter, t.x, t.y) * dir
        
        # Sample volume alpha at position p
        # [G] Assuming sample_volume_alpha is defined elsewhere
        sample_alpha = sample_volume_alpha(p)
        
        # Accumulate alpha using front-to-back blending
        payload.alpha += (1.0 - payload.alpha) * sample_alpha
        
        # Return true if marching should continue
        return payload.alpha < 0.9999 # nearly_one
    
    raymarching_iterator(org, dir, t_min, t_max, sampling_scale * step_size, sample_lambda)
    
    return payload.alpha
```

## Subtopics

### [Shadow Volume Representation (SVR)](4603-shadow-volume-representation-svr.md)
Shadow Volume Representation (SVR) is a sparse data structure that stores global opacity ($\alpha_g$) along rays from volume points to a light source, enabling efficient estimation of direct light contribution in participating media. It distinguishes itself from full-resolution ray integration by decoupling the storage resolution from the integration resolution, allowing interactive adjustment of the shadow map density while maintaining high-quality visibility estimates. Choose SVR when you need to balance shadow quality and performance by adjusting density independently of the rendering resolution. Link: [Shadow Volume Representation (SVR)](4603-shadow-volume-representation-svr.md).

### Transmittance Function Mapping
Transmittance Function Mapping (TFM) approximates the transmittance function using orthogonal basis functions (e.g., Fourier series) to calculate self-shadowing more efficiently than ray marching [S3, unofficial]. It generates a volume shadow map from the sun direction using orthographic projection, storing Fourier coefficients in a 512x512x3 R11G11B10 format (first 9 coefficients) [S3, unofficial]. During rendering, it reads Fourier coefficients and combines them linearly with basis functions to reconstruct optical depth and transmittance [S3, unofficial]. TFM efficiency is roughly equivalent to Ray Marching with 1-2 steps but provides more accurate results than 2-step RM, though it may produce banding artifacts [S3, unofficial]. Choose TFM when you need higher accuracy than low-step ray marching with similar performance, and can tolerate potential banding.

### Cloud Shadow Map Directional Consistency Fix
Cloud Shadow Map Directional Consistency Fix addresses incorrect cloud shadows for non-parallel lights by ensuring the cloud shadow map is traced along the view's first directional light, but consumers look it up in that light's own basis [S4, unofficial]. Previously, every consumer looked it up for each light in that light's own basis, causing a second, nonparallel sun to read the first one's optical depths as a displaced, wrongly oriented shadow [S4, unofficial]. For non-parallel lights, it marches the cloud layer toward the light using `cloud_far_transmittance` in the atmosphere passes, leaving meshes unshadowed by clouds for those lights [S4, unofficial]. Parallel lights continue to share the shadow map [S4, unofficial]. Choose this fix when rendering scenes with multiple non-parallel directional lights (e.g., multiple suns) to avoid shadow orientation artifacts.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Uses extra memory for storing the shadow volume [S2].
- **Measured Performance:** TFM efficiency is roughly equivalent to Ray Marching with 1-2 steps [S3, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Decouples storage resolution from integration resolution [S1, unofficial]; allows interactive adjustment of shadow map density [S1, unofficial]; TFM provides more accurate results than low-step RM [S3, unofficial].
- **Limitations:** Results in blurry shadows and dark images [S2]; TFM may produce banding artifacts [S3, unofficial]; uses extra memory [S2].
- **When to Use:** When self-shadowing in participating media is required and performance allows for precomputation or sparse storage.
- **When Not to Use:** When memory is constrained or when blurry shadows are unacceptable.

## Background
Shadow Volume Rendering was introduced as a method to incorporate complex lighting effects, such as volumetric shadows, into volume rendering [S2]. It was developed as an alternative to simple lighting models that assume light arrives at a sample without interacting with the rest of the volume [S2]. The technique is used in various volume rendering applications, including cloud rendering [S3, unofficial] and atmospheric scattering [S4, unofficial].

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S2] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S3] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S4] WIP: Nubis-based volumetric clouds, https://github.com/mate-h/bevy/pull/26, fetched 2026-10-27 (unofficial)
