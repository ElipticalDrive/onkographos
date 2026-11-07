# Next Event Estimation for Emissive Volumes

Also called: Emissive Media Rendering, Emissive Volume Rendering with Next Event Estimation (NEE), NEE for Emissive Volumes

## Overview
Next Event Estimation (NEE) for emissive volumes computes the direct lighting contribution from light-emitting media by sampling points directly on the emissive volume geometry rather than relying solely on volumetric scattering along the ray [S1, unofficial]. This method reduces noise in scenes containing bright emissive regions by ensuring that light sources are explicitly sampled, addressing the high variance inherent in pure path tracing through participating media [S1, unofficial]. It differs from analytic shading [Direct volume rendering with Phong shading](3686-direct-volume-rendering-with-phong-shading.md) by using statistical sampling for direct illumination rather than local gradient approximations, and from illumination caching [Illumination caching](7548-illumination-caching.md) by computing lighting dynamically per sample rather than pre-integrating it into 3D textures [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current ray segment (origin $\mathbf{o}$, direction $\vec{\omega}$, entry point $\mathbf{p}_{in}$, exit point $\mathbf{p}_{out}$), the emissive volume geometry (bounding shape or mesh), and the volume density/emission coefficients at sample points.
*   **Output:** The direct lighting term $L_{direct}(\mathbf{p})$ to be added to the shading calculation at sample point $\mathbf{p}$ along the ray.

**Data Structures:**
*   **Ray Segment:** Defined by origin $\mathbf{o}$ and direction $\vec{\omega}$.
*   **Intersection Data:** Entry point $\mathbf{p}_{entry}$ and exit point $\mathbf{p}_{exit}$ of the ray within the emissive volume's bounding shape. These are required to calculate the exact directional probability density function (PDF) [S1, unofficial].
*   **Light Source Representation:** The emissive volume itself acts as the light source. It is treated as a geometric primitive for sampling purposes.

**Steps:**
1.  **Sample a Point on the Emissive Volume:**
    Select a point $\mathbf{p}_L$ on the surface or within the volume of the emissive medium. This is done by sampling the geometry of the emissive volume. Let $p(\vec{x})$ be the PDF of sampling this point $\mathbf{p}_L$ in volume space [S1, unofficial].

2.  **Calculate Direction and Distance:**
    Compute the vector from the shading point $\mathbf{p}$ (on the primary ray) to the sampled light point $\mathbf{p}_L$:
    $$ \vec{d} = \mathbf{p}_L - \mathbf{p} $$
    Calculate the distance $r = \|\vec{d}\|$.
    Compute the normalized direction to the light:
    $$ \vec{\omega}_L = \frac{\vec{d}}{r} $$

3.  **Compute Solid Angle PDF:**
    Determine the PDF of the direction $\vec{\omega}_L$ in solid angle space, denoted $p(\vec{\omega})$. This depends on the sampling strategy used for the emissive volume geometry [S1, unofficial].

4.  **Convert to Volume PDF (MIS Weighting):**
    To combine this sampling strategy with the standard volumetric path tracing (which samples along the ray direction), a Multiple Importance Sampling (MIS) weight is required. The MIS weight is computed as:
    $$ \text{weight} = \text{MIS}(r^{-2}p(\vec{\omega}), p(\vec{x})) $$
    where $r^{-2}p(\vec{\omega})$ represents the volume PDF derived from the solid angle PDF $p(\vec{\omega})$ at distance $r$, and $p(\vec{x})$ is the standard volume PDF associated with the ray's sampling distribution [S1, unofficial].

    **Critical Implementation Detail:**
    Simply converting the solid angle PDF to a volume PDF using the differential element $r^{-2}\frac{\mathrm{dV}}{\mathrm{d\Omega}}$ yields incorrect results if not handled consistently with the emission evaluation [S1, unofficial]. The term $r^{-2}p(\vec{\omega})dV$ is a non-normalisable PDF because the integral $\int_V r^{-2} p(\vec{\omega}) \mathrm{dV}$ expands to $\int_V p(\vec{\omega})\mathrm{\sin\theta dr d\theta d\phi}$, which does not converge to 1 in the same manner as a standard normalized PDF [S1, unofficial]. Therefore, the MIS weight must be calculated carefully to ensure consistency [S1, unofficial].

5.  **Evaluate Emission:**
    Evaluate the emission $E(\mathbf{p}_L)$ at the sampled point $\mathbf{p}_L$. To ensure consistency with the direction sampling used in the MIS calculation, the emission along the entire ray segment from $\mathbf{p}$ to $\mathbf{p}_L$ must be evaluated (attenuated by transmittance) [S1, unofficial].

6.  **Compute Direct Lighting Contribution:**
    The direct lighting contribution $L_{direct}$ is computed using the standard NEE formula, adjusted for the volume medium:
    $$ L_{direct}(\mathbf{p}) = \frac{E(\mathbf{p}_L) \cdot T(\mathbf{p}, \mathbf{p}_L) \cdot \text{weight}}{p(\vec{x})} $$
    where $T(\mathbf{p}, \mathbf{p}_L)$ is the transmittance from $\mathbf{p}$ to $\mathbf{p}_L$, accounting for absorption and scattering along the path [S1, S2, unofficial].

7.  **Shadow Ray (Optional but Recommended):**
    Cast a shadow ray from $\mathbf{p}$ to $\mathbf{p}_L$ with interval $[0.001, r - 0.001]$ to check for occlusion by other opaque or absorbing objects [S2, unofficial]. If occluded, the contribution is zero.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_event_estimation_emissive_volume(p, ray_origin, ray_direction, emissive_volume, volume_density_func, transmittance_func):
    """
    Computes direct lighting from an emissive volume using Next Event Estimation.
    
    Args:
        p: Current shading point on the ray (Vector3).
        ray_origin: Origin of the primary ray (Vector3).
        ray_direction: Direction of the primary ray (Vector3).
        emissive_volume: Object representing the emissive volume geometry.
        volume_density_func: Function returning density at a point.
        transmittance_func: Function returning transmittance between two points.
        
    Returns:
        Direct lighting contribution (RGB).
    """
    
    # 1. Sample a point on the emissive volume
    # Note: The specific sampling method depends on the geometry of the emissive volume.
    # For a bounding box or mesh, use standard geometric sampling.
    p_light, pdf_volume = emissive_volume.sample_point()
    
    # 2. Calculate direction and distance
    d = p_light - p
    r = d.norm()
    
    # Avoid division by zero or self-intersection
    if r < 0.001:
        return RGB(0, 0, 0)
        
    omega_L = d / r
    
    # 3. Get solid angle PDF
    # This depends on the sampling strategy. For uniform sampling over a sphere, 
    # it would be 1/(4*pi). For area sampling, it relates to the projected area.
    # Here we assume pdf_volume is the volume PDF. We need the solid angle PDF p(omega).
    # If sampling was done via area, p(omega) = pdf_volume * r^2 / cos(theta) ? 
    # The source indicates MIS(r^-2 * p(omega), p(x)). 
    # Let's assume we have p(omega) available from the sampler or derived.
    # For simplicity, if we sampled area A, p(omega) = p_area * r^2 / (A * cos_theta) ?
    # The source emphasizes the MIS weight calculation.
    
    # Let's assume we have p_omega from the sampler's context or derived from p_volume
    # p_volume = p_omega * sin(theta) / r^2 ? 
    # Actually, the source says: MIS(r^-2 * p(omega), p(x))
    # So we need p(omega). If the sampler returns p_volume, we might need to invert.
    # However, the source notes that simply converting is tricky.
    # Let's assume the sampler provides p_omega directly or we compute it.
    
    # For a generic emissive volume, let's assume we have p_omega.
    # If not, and we only have p_volume, we must be careful.
    # The source says: "MIS is computed via MIS(r^-2 * p(omega), p(x))"
    # And "p(x) is the standard volume PDF".
    
    # Let's assume p_omega is available.
    p_omega = ... # Derived from sampling strategy, e.g., 1/(4*pi) for uniform sphere
    
    # 4. Compute MIS Weight
    # r^-2 * p(omega)
    pdf_volume_from_omega = (1.0 / (r * r)) * p_omega
    
    # p(x) is the PDF of the current ray's sampling strategy.
    # In standard path tracing, this is often 1 for the direction, but for NEE,
    # we are comparing two strategies: sampling the light vs sampling the volume.
    # The source implies p(x) is the "standard volume PDF".
    # In the context of the ray, p(x) might be 1 if we are just evaluating the light term.
    # However, MIS usually balances two PDFs.
    # Let's assume p_x is the PDF of the current path's direction sampling.
    # If we are just adding a light term, p_x might be 1.
    
    # The source says: MIS(r^-2 * p(omega), p(x))
    # Let's use balance heuristic.
    weight = balance_heuristic(pdf_volume_from_omega, p_volume)
    
    # 5. Evaluate Emission
    # Emission at the light point
    emission = emissive_volume.get_emission(p_light)
    
    # 6. Transmittance
    # Transmittance from p to p_light
    T = transmittance_func(p, p_light)
    
    # 7. Compute Contribution
    # L_direct = (emission * T * weight) / p_volume
    # Note: The source says MIS(r^-2 * p(omega), p(x)).
    # The standard NEE formula is: L = (f * T * E) / p_light
    # Here, the "light" is the volume.
    
    # If we use MIS, the estimator is:
    # L = (emission * T * weight) / p_volume
    
    if weight == 0 or p_volume == 0:
        return RGB(0, 0, 0)
        
    direct_light = (emission * T * weight) / p_volume
    
    return direct_light

def balance_heuristic(pdf1, pdf2):
    """Balance heuristic for MIS."""
    if pdf1 == 0 and pdf2 == 0:
        return 0
    return pdf1 / (pdf1 + pdf2)
```

**Parameters:**
*   **Ray Interval for Shadow Ray:** $[0.001, r - 0.001]$ to avoid self-intersection and precision issues [S2, unofficial].
*   **MIS Strategy:** Balance heuristic is implied by the notation $\mathrm{MIS}(f, g)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-normalisable PDF:** The term $r^{-2}p(\vec{\omega})dV$ is not a normalisable PDF. Integrating it over the volume does not yield 1. This requires careful handling in the MIS weight calculation to avoid incorrect results [S1, unofficial].
*   **Inconsistency with Emission Evaluation:** The emission must be evaluated along the entire ray segment to ensure consistency with the direction sampling used in the MIS calculation [S1, unofficial].
*   **Performance:** Integrating the volume PDF over the ray length is significantly slower (20x-40x) due to required ray casts and raises OptiX warnings in CUDA modes [S1, unofficial].
*   **Platform Discrepancy:** In `llvm_*` modes, the implementation using NEE is approximately 5x slower than not using NEE. In `cuda_*` modes, it is approximately 1.5x slower [S1, unofficial].
*   **Intersection Handling:** Different handling of ray intersections by Embree (llvm) vs OptiX (cuda) can lead to different results for ray entry and exit points, potentially causing incorrect results in `llvm_*` modes if not fixed [S1, unofficial].

## Subtopics

### Path Tracing with Next Event Estimation
A general path tracing algorithm that includes direct lighting estimation via Next Event Estimation (NEE) [S2, unofficial]. It recursively traces rays to compute pixel color. The base case returns black if depth is 0. It samples points on lights, computes shadow rays, and combines emitted, direct, and indirect lighting [S2, unofficial]. This is the broader context in which NEE for emissive volumes is often implemented, treating the emissive volume as a light source [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method adds overhead due to the need to sample the emissive volume geometry and potentially cast shadow rays. In `llvm_*` modes, it is approximately 5x slower than not using NEE. In `cuda_*` modes, it is approximately 1.5x slower [S1, unofficial].
*   **Memory Complexity:** Requires storing the emissive volume geometry and potentially pre-computed structures for efficient sampling. No significant additional memory compared to standard volume rendering.
*   **Measured Performance:** The implementation reduces noise for the same sample count, allowing for faster convergence in scenes with bright emissive volumes [S1, unofficial].
*   **Strengths:** Reduces noise in scenes with emissive media by explicitly sampling light sources. More efficient than pure volumetric scattering for direct lighting.
*   **Limitations:** Slower than non-NEE approaches in some implementations due to geometric sampling overhead. Sensitive to precision issues in ray intersection calculations, particularly in `llvm_*` modes [S1, unofficial].
*   **When to Use:** When rendering scenes with bright emissive volumes where noise from pure path tracing is unacceptable.
*   **When Not to Use:** When performance is critical and the emissive volume is not a dominant light source, or when using hardware that does not support efficient geometric sampling (e.g., certain `llvm_*` modes) [S1, unofficial].

## Background
Next Event Estimation for emissive volumes was implemented in the Mitsuba 3 renderer as part of a feature pull request [S1, unofficial]. The implementation was developed by contributors Microno95 and dvicini, with discussions occurring in June 2023 [S1, unofficial]. The method addresses the challenge of efficiently sampling direct lighting from participating media, which is a known difficulty in volumetric path tracing [S1, unofficial]. The implementation notes highlight the complexities of converting solid angle PDFs to volume PDFs and the importance of consistency in MIS weighting [S1, unofficial].

## Sources
[S1] [✨ feature] Emissive Volume Rendering, https://github.com/mitsuba-renderer/mitsuba3/pull/678, fetched 2026-10-26 (unofficial)
[S2] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
