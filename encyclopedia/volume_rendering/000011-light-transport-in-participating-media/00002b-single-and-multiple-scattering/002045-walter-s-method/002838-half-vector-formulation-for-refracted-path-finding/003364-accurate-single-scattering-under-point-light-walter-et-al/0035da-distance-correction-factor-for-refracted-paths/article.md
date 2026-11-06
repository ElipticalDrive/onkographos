# Distance Correction Factor for Refracted Paths

Also called: D factor for refracted paths, Distance Correction Factor D for Refracted Paths, Geometric spreading factor for refraction

## Overview
The Distance Correction Factor ($D$) accounts for geometric spreading and refraction effects in the contribution formula for light paths involving refractive boundaries in participating media [S1]. It ensures that the computed radiance matches reference solutions by correctly scaling the intensity based on the solid angle subtended by the light source as seen from the scattering point, adjusted for the refractive index [S1]. It is distinct from simple inverse-square distance laws because it incorporates the Jacobian of the refraction mapping at the boundary [S1].

## How it works
**Inputs:**
*   $\mathbf{L}$: Position of the point light source.
*   $\mathbf{V}$: Position of the camera sample or volume scattering event.
*   $\mathbf{P}$: Position of the refraction point on the boundary surface.
*   $\hat{\omega}_L$: Unit vector from $\mathbf{P}$ to $\mathbf{L}$ ($\frac{\mathbf{L}-\mathbf{P}}{\|\mathbf{L}-\mathbf{P}\|}$).
*   $\hat{\omega}_V$: Unit vector from $\mathbf{P}$ to $\mathbf{V}$ ($\frac{\mathbf{V}-\mathbf{P}}{\|\mathbf{V}-\mathbf{P}\|}$).
*   $\hat{\mathbf{N}}_g$: Geometric normal at $\mathbf{P}$.
*   $\hat{\mathbf{N}}_s$: Shading normal at $\mathbf{P}$.
*   $\eta$: Refractive index of the medium relative to the outside (typically 1.5) [S1].
*   $I_e$: Point light intensity [S1].
*   $F$: Fresnel factor at $\mathbf{P}$ [S1].
*   $A$: Volume attenuation (integral of $\sigma_s$ along the segment) [S1].
*   $\rho$: Volume scattering phase function (or BRDF $\times$ cosine if $\mathbf{V}$ is a surface) [S1].

**Outputs:**
*   $D$: The distance correction factor.
*   Contribution: $\frac{I_e F A \rho}{D}$ [S1].

**Data Structures:**
*   Vectors $\mathbf{L}, \mathbf{V}, \mathbf{P}$ are 3D floats.
*   Normals $\hat{\mathbf{N}}_g, \hat{\mathbf{N}}_s$ are unit 3D floats.
*   Derivatives $\hat{\mathbf{N}}'_s$ are 3D floats representing the change in shading normal with respect to perturbations in $\hat{\omega}_V$.

**Steps:**

1.  **Check for No Refraction:**
    If $\eta = 1$, return $D = \|\mathbf{L} - \mathbf{V}\|$ [S1].

2.  **Check for Normal Incidence without Shading Normals:**
    If $\hat{\omega}_L = \hat{\mathbf{N}}_g = \hat{\mathbf{N}}_s$ (normal incidence and shading normal equals geometric normal), return $D = (\|\mathbf{V} - \mathbf{P}\| + \eta \|\mathbf{L} - \mathbf{P}\|)^2$ [S1].

3.  **Special Case: No Shading Normals ($\hat{\mathbf{N}}_s = \hat{\mathbf{N}}_g$):**
    Compute distances:
    $$d_V = \|\mathbf{V} - \mathbf{P}\|$$
    $$d_L = \|\mathbf{L} - \mathbf{P}\|$$
    Return:
    $$D = (d_V + \eta d_L) \left( \frac{|\hat{\omega}_L \cdot \hat{\mathbf{N}}_g|}{|\hat{\omega}_L \cdot \hat{\mathbf{N}}_g|} d_V + \frac{|\hat{\omega}_V \cdot \hat{\mathbf{N}}_g|}{|\hat{\omega}_L \cdot \hat{\mathbf{N}}_g|} \eta d_L \right)$$
    Note: The term $\frac{|\hat{\omega}_L \cdot \hat{\mathbf{N}}_g|}{|\hat{\omega}_L \cdot \hat{\mathbf{N}}_g|}$ simplifies to 1, but is preserved as per the source notation [S1].

4.  **General Case: With Shading Normals:**
    Compute derivatives of the shading normal $\hat{\mathbf{N}}_s$ with respect to perturbations in $\hat{\omega}_V$. Let $\hat{\mathbf{N}}'_s$ be the derivative of $\hat{\mathbf{N}}_s$ [S1].

    Compute intermediate vectors and scalars:
    $$d_V = \mathbf{V} - \mathbf{P}$$
    $$d_L = \mathbf{L} - \mathbf{P}$$
    
    Compute $\mathbf{P}'$:
    $$\mathbf{P}' = d_V \left( \partial_V - \frac{\partial_V \cdot \hat{\mathbf{N}}_g}{\partial_V \cdot \hat{\mathbf{N}}_g} \hat{\omega}_V \right)$$
    [G] Note: The source contains a likely typo in the denominator of the fraction in Eq 21, shown as $\partial_V \cdot \hat{\mathbf{N}}_g / \partial_V \cdot \hat{\mathbf{N}}_g$, which equals 1. The implementation should likely use a projection operator or the specific derivative definition from Igehy [1999] for ray differentials. However, following the source strictly:
    $$\mathbf{P}' = d_V \left( \partial_V - \hat{\omega}_V \right)$$
    [G] $\partial_V$ represents the derivative of the position $\mathbf{V}$ with respect to the perturbation in $\hat{\omega}_V$. In the context of ray differentials, this is typically computed by perturbing $\hat{\omega}_V$ in two orthogonal directions perpendicular to $\hat{\omega}_V$ and computing the resulting change in $\mathbf{V}$. If $\mathbf{V}$ is fixed (camera ray origin), $\partial_V$ might be zero or derived from the camera ray differential. The source text is ambiguous on the definition of $\partial_V$ other than it being part of the derivative chain. We assume $\partial_V$ is the derivative of the vector $\mathbf{V}$ with respect to the angular perturbation. If $\mathbf{V}$ is a point in space fixed relative to the camera, and we perturb the ray direction $\hat{\omega}_V$ at $\mathbf{P}$, $\mathbf{V}$ does not change, so $\partial_V = 0$. However, the formula suggests $\partial_V$ is a vector. Let us assume the standard ray differential approach where we track the derivative of the ray origin/position. If $\mathbf{V}$ is the camera position, it is constant. If $\mathbf{V}$ is a scattering point, it depends on the path. The source equation 21 is: $\mathbf{P}' = d_V ( \partial_V - \frac{\partial_V \cdot \hat{\mathbf{N}}_g}{\partial_V \cdot \hat{\mathbf{N}}_g} \hat{\omega}_V )$. Given the typo, and standard ray differentials, $\mathbf{P}'$ is likely the derivative of the point $\mathbf{P}$ on the surface. Let us proceed with the calculation of $\mu$ and $\mu'$ as defined.

    $$\mu = \hat{\omega}_L \cdot \hat{\mathbf{N}}_s + \eta \hat{\omega}_V \cdot \hat{\mathbf{N}}_s$$
    
    Compute $\mu'$:
    $$\mu' = \left( \eta^2 \frac{\partial \omega \cdot \hat{\mathbf{N}}_s}{\partial \omega \cdot \hat{\mathbf{N}}_s} + \eta \right) (\hat{\omega}'_V \cdot \hat{\mathbf{N}}_s + \hat{\omega}_V \cdot \hat{\mathbf{N}}'_s)$$
    [G] The term $\frac{\partial \omega \cdot \hat{\mathbf{N}}_s}{\partial \omega \cdot \hat{\mathbf{N}}_s}$ is likely a typo for a derivative of the dot product or a scalar factor. The source quotes it exactly. We treat it as 1 if interpreted literally, or as a placeholder for the derivative of the dot product term. Given the context of ray differentials, this term likely relates to the derivative of the cosine term. However, without a clear definition, we follow the source structure. Let $K = \eta^2 \cdot 1 + \eta$ if the fraction is 1.
    
    Compute $\hat{\omega}'_V$:
    $$\hat{\omega}'_V = \eta \hat{\omega}'_V + \mu' \hat{\mathbf{N}}_s + \mu \hat{\mathbf{N}}'_s$$
    [G] The source uses $\hat{\omega}'_V$ on both sides. This implies an update or a specific notation where the RHS $\hat{\omega}'_V$ is the derivative of the incoming vector. Let $\hat{\omega}'_{V, in}$ be the derivative of $\hat{\omega}_V$. Then $\hat{\omega}'_{V, out} = \eta \hat{\omega}'_{V, in} + \mu' \hat{\mathbf{N}}_s + \mu \hat{\mathbf{N}}'_s$. The source then uses this result in the next step.
    
    Compute $\phi'_L$ (likely $\hat{\omega}'_L$):
    $$\hat{\omega}'_L = - \eta \hat{\omega}'_L + \mu' \hat{\mathbf{N}}_s + \mu \hat{\mathbf{N}}'_s$$
    [G] The source labels this $\phi'_L$ but uses it in the context of $\mathbf{L}'$. It is likely $\hat{\omega}'_L$. The RHS $\hat{\omega}'_L$ is likely the derivative of the vector from $\mathbf{P}$ to $\mathbf{L}$ if $\mathbf{L}$ moves, or 0 if $\mathbf{L}$ is fixed. If $\mathbf{L}$ is fixed, $\hat{\omega}'_L$ comes from the change in $\mathbf{P}$. The source equation is recursive. We assume the RHS is the derivative of the unrefracted direction.
    
    Compute $\mathbf{L}'$:
    $$\mathbf{L}' = \mathbf{P}' - (\mathbf{P}' \cdot \hat{\omega}_L) \hat{\omega}_L + d_L \hat{\omega}'_L$$
    [G] This computes the derivative of the vector $\mathbf{L} - \mathbf{P}$ projected into the plane perpendicular to $\hat{\omega}_L$ plus the change due to direction.
    
    Compute derivatives in two orthogonal directions perpendicular to $\hat{\omega}_V$ and $\hat{\omega}_L$ to get $\mathbf{L}'_\perp$ and $\mathbf{L}'_\parallel$ [S1].
    
    Return:
    $$D = \|\mathbf{L}'_\perp \times \mathbf{L}'_\parallel\|$$

**Parameters:**
*   $\eta$: Typical value 1.5 for homogeneous media [S1].

**Edge Cases and Pitfalls:**
*   **Focal Points/Cusps:** Radiance can grow arbitrarily large near focal points or cusps in volume caustics, leading to spike noise [S1].
*   **Solution:** Clamp $D$ to a minimum threshold to reduce noise, at the cost of some energy loss [S1].
*   **Derivatives:** Requires computing derivatives of shading normals $\hat{\mathbf{N}}_s$, which can be computed for triangles with interpolated normals [S1].

## Operating characteristics
*   **Complexity:** Computing $D$ is described as "not difficult to code" once the equations are known, requiring the computation of shading normal derivatives [S1].
*   **Performance:** The method is used in scenes with varying complexity (e.g., teapot with 12 boundaries, 15.3s; cuboctahedron with 20 boundaries, 13.9s; amber with 36 boundaries, 19.2s) for 512x512 images with 64 samples per pixel [S1].
*   **Strengths:** Provides visually correct results matching reference solutions for single scattering in refractive media [S1].
*   **Limitations:** Can produce spike noise near caustic focal points if not clamped [S1].

## Background
The method is part of the technique for accurate single scattering under point lights in refractive media with triangle mesh boundaries, introduced by Walter et al. [S1]. It builds on ray differential techniques from Igehy [1999] to compute the solid angle of the light source as seen from the scattering point through the refractive boundary.

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
