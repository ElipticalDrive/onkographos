# Indirect Single Scattering via Precomputed Integrals

Also called: Ambient single scattering, Liss computation

## Overview
This method approximates the indirect single scattering contribution in participating media by precomputing angular integrals of the single-scattering term into lookup textures, allowing real-time evaluation via texture sampling [S1, unofficial]. It differs from direct single scattering methods by integrating the scattered light over the hemisphere of incoming directions weighted by surface reflectance, rather than computing scattering along a single view ray [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Point position $P$, surface normal $\mathbf{n}$, light direction $\mathbf{L}$, view direction $\mathbf{V}$, distance to light $l = \|\mathbf{L}\|$, medium extinction coefficient $k_t$, surface diffuse reflectivity $k_d$, specular reflectivity $k_s$, shininess exponent $n$.
- **Outputs:** Indirect single scattering radiance $L_{iss}(P)$ added to the pixel color.

**Data Structures:**
- `gamma_lambert`: 1D texture storing $\Gamma_L(\lambda)$ values.
- `gamma_phong`: 2D texture storing $\Gamma_P(\phi', \lambda)$ values.
- `integralRange`: Uniform vector $(x, y, z, w)$ defining the affine mapping for texture lookups: $value = texture.r \cdot range.y + range.x$ for Lambertian, and $texture.r \cdot range.w + range.z$ for Phong.

**Formulas:**
The indirect single scattering radiance is computed separately for Lambertian and Phong components [S1, unofficial].

For Lambertian surfaces:
$$ L_{iss}(P) = \frac{k_t^2 \Omega_{IS}}{4\pi} \frac{2\pi k_d \cos(\phi_n)}{k_t l} \Gamma_L(k_t l) $$
where $\Gamma_L(\lambda)$ is:
$$ \Gamma_L(\lambda) = \int_0^{\pi/2} \int_{\phi-\pi/2}^{\pi/2} \frac{\cos(\alpha-\phi)+1}{\pi} p(\alpha_+) e^{-\lambda \cos(\alpha)} \cos(\phi) \, d\alpha \, d\phi $$
Here, $\phi_n$ is the angle of the normal vector in a spherical coordinate system centered on $P$, and $\Omega_{IS}$ is the solid angle of the light source [S1, unofficial].

For Phong surfaces (if specular component exists):
$$ L_{iss}(P) = \frac{k_t^2 \Omega_{IS}}{4\pi} \frac{k_s}{k_t l} \Gamma_P(\phi', k_t l) $$
where $\Gamma_P(\phi', \lambda)$ is:
$$ \Gamma_P(\phi', \lambda) = \int_0^{\pi/2} \int_0^{\pi} \int_{\phi'-\pi/2}^{\pi/2} \frac{\cos(\alpha-\phi')+1}{\pi} p(\alpha_+) e^{-\lambda \cos(\alpha)} \cos^n(\phi) \, d\alpha \, d\phi \, d\theta $$
$\phi'$ is the angle between the light direction and the view direction [S1, unofficial].

**Steps:**
1. Compute distance to light: $l = \|\mathbf{L}\|$.
2. Compute Lambertian term:
   - Scale distance by extinction: $\lambda = k_t l$.
   - Lookup $\Gamma_L(\lambda)$ from `gamma_lambert` using $\lambda$ (scaled by 10.0 in the source implementation) and `integralRange` to get the value.
   - Compute $L_{iss, Lambert} = \frac{6.2831853 \cdot k_d \cos(\phi_n) \cdot \Gamma_L}{k_t l}$.
   - Multiply by material diffuse color.
3. Compute Phong term (if $k_s > 0$):
   - Compute $\phi'$ from $\mathbf{L}$ and $\mathbf{V}$.
   - Lookup $\Gamma_P(\phi', \lambda)$ from `gamma_phong` using $\lambda$ and $\phi'$ and `integralRange`.
   - Compute $L_{iss, Phong} = \frac{k_s \cdot \Gamma_P}{k_t l}$.
   - Multiply by material specular color.
4. Combine with direct illumination and apply attenuation:
   - $Color_{final} = Color_{direct} + k_t \cdot (L_{iss, Lambert} + L_{iss, Phong}) \cdot e^{-k_t \cdot dist} \cdot Attenuation$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_iss(light_dir, normal, view_dir, light_dist, kt, kd, ks, n, integral_range, gamma_lambert_tex, gamma_phong_tex):
    # light_dir is normalized vector from point to light
    # normal is surface normal
    # view_dir is normalized vector from point to camera
    # light_dist is distance from point to light source
    # kt is extinction coefficient
    # kd is diffuse reflectivity
    # ks is specular reflectivity
    # n is shininess exponent
    
    l = light_dist
    factors_r = l * kt  # lambda = kt * l
    
    # Lambertian term
    lambert_term = dot(normal, light_dir)
    if lambert_term > 0.0:
        # Lookup Gamma_L
        # Source uses factors.r / 10.0 as texture coordinate
        tex_coord_l = factors_r / 10.0
        gamma_l = texture1D(gamma_lambert_tex, tex_coord_l).r
        # Apply integral range
        gamma_l = gamma_l * integral_range.y + integral_range.x
        
        # Liss_L = (kt^2 * Omega_IS / 4pi) * (2pi * kd * cos(phi_n) / (kt * l)) * Gamma_L
        # Note: The source implementation simplifies constants. 
        # 6.2831853 is approx 2*pi. The term 2pi * kd * cos(phi_n) / (kt * l) is applied.
        # The source code: resultat.a = 6.2831853 * lambertTerm * facteurs.g / facteurs.r
        # where facteurs.g is gamma_l and facteurs.r is kt*l
        iss_lambert_val = (2.0 * 3.14159265 * lambert_term * gamma_l) / factors_r
        
        # Multiply by diffuse color (assumed passed or global)
        # resultat.rgb = gl_FrontMaterial.diffuse.rgb * resultat.a
        iss_lambert_rgb = kd * iss_lambert_val
    else:
        iss_lambert_rgb = 0.0

    # Phong term
    iss_phong_rgb = 0.0
    if ks > 0.0:
        # Compute phi' (angle between light and view)
        # In the source, dre = max(dot(R, E), 0.0) where R is reflection of L, E is view
        # This is equivalent to computing the angle between L and V via reflection
        # However, the formula uses phi' directly. The source uses 'dre' as a proxy for the angle component in the texture lookup.
        # Specifically, the source uses 'dre' as the second coordinate for gamma_phong.
        # dre = max(dot(reflect(-light_dir, normal), -view_dir), 0.0)
        # Note: The source code calculates dre based on reflection vector.
        R = reflect(-light_dir, normal)
        E = -view_dir # Assuming view_dir is from point to camera, eye vector is from camera to point? 
                      # Source: vec3 E = normalize(eyeVec); eyeVec is likely camera to point.
                      # If view_dir is point to camera, E = -view_dir.
        dre = max(dot(R, -view_dir), 0.0)
        
        # Lookup Gamma_P
        # Source uses facteurs.g = facteurs.r / 10.0 (lambda) and facteurs.b = dre
        tex_coord_p = vec2(factors_r / 10.0, dre)
        gamma_p = texture2D(gamma_phong_tex, tex_coord_p).r
        # Apply integral range
        gamma_p = gamma_p * integral_range.w + integral_range.z
        
        # Liss_P = (kt^2 * Omega_IS / 4pi) * (ks / (kt * l)) * Gamma_P
        # Source: resultat.a = texture2D... * range.w + range.z
        # resultat.rgb += gl_FrontMaterial.specular.rgb * resultat.a / facteurs.r
        iss_phong_val = gamma_p / factors_r
        iss_phong_rgb = ks * iss_phong_val

    return iss_lambert_rgb, iss_phong_rgb
```

**Parameters:**
- $l$: Distance from point $P$ to the light source.
- $k_d$: Diffuse reflectivity (albedo).
- $k_s$: Specular reflectivity.
- $n$: Shininess exponent.
- $\phi_n$: Angle of the normal vector in the spherical coordinate system centered on $P$.

**Edge Cases and Pitfalls:**
- The method assumes the distance to scattering points $d_{\phi,\theta} = \infty$ to avoid complexity, which is a coarse approximation [S1, unofficial].
- The number of instructions in the shader can exceed GPU capacity if direct and indirect computations are combined in a single pass [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for the lookup and arithmetic operations, assuming texture lookups are constant time.
- **Memory Complexity:** Depends on the resolution of `gamma_lambert` and `gamma_phong` textures.
- **Strengths:** Real-time performance due to precomputed integrals and texture lookups.
- **Limitations:** Approximation error due to the infinite distance assumption for scattering points. Limited to single scattering.

## Background
This method was introduced in "Real Time Single Scattering Effects" [S1, unofficial]. It builds on the analytical formulation of single scattering in participating media and uses precomputed integrals to accelerate the evaluation of indirect single scattering.

## Sources
[S1] Real Time Single Scattering Effects, https://hal.science/hal-00622215v1, fetched 2026-10-26 (unofficial)
