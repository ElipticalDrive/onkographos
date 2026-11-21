# Spherical Harmonic Volume Rendering

Also called: Advanced Light Material Interaction for DVR, SH-based light material interaction, Spherical Harmonic (SH) Volume Rendering with Advanced Material Interaction

## Overview
Spherical Harmonic Volume Rendering is a heuristic method for simulating advanced light-material interactions in interactive direct volume rendering (DVR) by incorporating complex material functions (reflectance and scattering) and colored area light sources using a unified Spherical Harmonic (SH) basis representation [S1]. Unlike slice-based techniques that are limited to headlights or cone-shaped scattering approximations, this approach supports arbitrary light directions and full-sphere scattering, enabling the simulation of environmental effects like in-scattering and color bleeding [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Volume data $\tau(\mathbf{x})$ (scalar field), material function $M(\mathbf{x}, \omega_i, \omega_o)$ represented in SH coefficients, incident radiance $L_i(\mathbf{x}, \omega_i)$ from area light sources, emission $EM(\mathbf{x}, \omega_o)$.
*   **Outputs:** Radiance $L(\mathbf{x}, \omega_o)$ leaving position $\mathbf{x}$ in direction $\omega_o$.

**Data Structures**
*   **SH Coefficients:** A vector of size $(L+1)^2$ containing the coefficients $c_{lm}$ for the spherical harmonic basis. For the standard implementation, $L=3$, resulting in 16 coefficients [S1].
*   **Material Function:** Defined over the whole sphere $\Omega$ (not just the hemisphere) because volume data lacks an inherent surface normal [S1].

**Mathematical Formulation**
The radiance $L(\mathbf{x}, \omega_o)$ leaving a position $\mathbf{x}$ in direction $\omega_o$ is defined as [S1]:

$$
L(\mathbf{x}, \omega_o) = EM(\mathbf{x}, \omega_o) + \int_{\Omega} M(\mathbf{x}, \omega_i, \omega_o) \cdot L_i(\mathbf{x}, \omega_i) d\omega_i
$$

To compute this efficiently, functions on the sphere are represented using Spherical Harmonic basis functions $Y_{lm}(\vartheta, \phi)$ [S1]. An infinite series representation of a function $f(\vartheta, \phi)$ is:

$$
f(\vartheta, \phi) = \sum_{l=0}^{\infty} \sum_{m=-l}^{l} c_{lm} Y_{lm}(\vartheta, \phi)
$$

The basis functions $Y_{lm}$ use complex exponential functions for latitude variations and associated Legendre polynomials for longitude variations [S1]. For practical implementation, the representation is bandlimited to a maximum order $L$:

$$
\tilde{f}(\vartheta, \phi) = \sum_{l=0}^{L} \sum_{m=-l}^{l} c_{lm} Y_{lm}(\vartheta, \phi)
$$

The number of summands is $(L+1)^2$ [S1]. The SH coefficients $c_j$ (where $j$ indexes the basis function $Y_j$) are computed via least square projection:

$$
c_j = \int_{S} Y_j(s) f(s) ds
$$

Due to the orthonormality of the SH basis functions, the integration of two SH-projected functions $\tilde{f}$ and $\tilde{g}$ with coefficients $c_i$ and $d_i$ respectively simplifies to a dot product:

$$
I(\tilde{f}, \tilde{g}) = \sum_{i} c_i d_i
$$

**Parameters**
*   **Maximum SH Order ($L$):** Set to 3 for practical reasons, resulting in 16 coefficients [S1].

**Algorithm Steps**
1.  **Precompute Material SH Coefficients:** For each voxel (or material category), compute the SH coefficients $c_{lm}$ representing the material function $M$. This involves projecting the material's reflectance/scattering properties onto the SH basis [S1].
2.  **Project Incident Light:** Represent the incident radiance $L_i(\mathbf{x}, \omega_i)$ from colored area light sources as SH coefficients $d_{lm}$ [S1].
3.  **Compute Interaction Integral:** Calculate the local light-material interaction term by taking the dot product of the material SH coefficients and the light SH coefficients: $\sum c_i d_i$ [S1].
    *   *Note:* As volume data lacks a surface normal, this modulation is **not** weighted by a clamped cosine term ($\cos \theta$) [S1].
4.  **Add Emission:** Add the emission term $EM(\mathbf{x}, \omega_o)$ to the result from Step 3 to obtain the final radiance $L(\mathbf{x}, \omega_o)$ [S1].
5.  **Render:** Use the computed radiance in the standard DVR compositing loop (e.g., front-to-back or back-to-front) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonic_volume_rendering(volume_data, light_sources, max_order=3):
    """
    Renders volume data using SH-based light material interaction.
    
    Args:
        volume_data: 3D array of scalar values (density/opacity).
        light_sources: List of light source definitions (direction, color, intensity).
        max_order: Maximum SH order L (default 3).
        
    Returns:
        Image buffer.
    """
    # Number of SH coefficients for order L
    num_coeffs = (max_order + 1) ** 2
    
    # Precompute SH coefficients for the material function M
    # M is defined over the whole sphere.
    # In a real implementation, this might be done per-voxel or per-material region.
    # Here we assume a uniform material for simplicity, or precomputed per-voxel lookup.
    material_sh_coeffs = compute_material_sh_coefficients(volume_data, max_order)
    
    # Project incident light Li into SH domain
    # Li depends on position x and direction omega_i
    # For area lights, this integration is over the sphere
    light_sh_coeffs = compute_light_sh_coefficients(light_sources, max_order)
    
    # Compute the interaction integral using dot product of SH coefficients
    # I = sum(c_i * d_i)
    interaction_integral = dot_product(material_sh_coeffs, light_sh_coeffs)
    
    # Add emission (EM) if present
    # EM is typically a function of position and outgoing direction
    # For this heuristic model, we assume EM is handled separately or is zero
    emission = get_emission(volume_data)
    
    # Final radiance
    radiance = emission + interaction_integral
    
    # Perform standard DVR compositing using the computed radiance
    image = perform_dvr_compositing(volume_data, radiance)
    
    return image

def compute_material_sh_coefficients(volume_data, max_order):
    """
    Computes SH coefficients for the material function M.
    M represents reflectance and scattering.
    """
    # The material function M(x, omega_i, omega_o) is projected onto SH basis.
    # Since M is defined over the whole sphere, no hemisphere masking is applied.
    # No cosine weighting is applied because volume data lacks surface normals.
    
    coeffs = [0.0] * ((max_order + 1) ** 2)
    
    # Numerical integration over the sphere to compute coefficients
    # c_j = integral(Y_j(s) * f(s) ds)
    # This step requires sampling the material function over the sphere
    # and projecting onto each basis function Y_j.
    
    # Implementation detail: Use quadrature points on the sphere.
    # For each basis function j:
    for j in range((max_order + 1) ** 2):
        integral_sum = 0.0
        for sample_point in sphere_quadrature_points():
            # Evaluate material function at sample point
            # M depends on incoming and outgoing directions, 
            # but for the interaction term, we project the angular dependence.
            material_value = evaluate_material_function(sample_point)
            
            # Evaluate basis function Y_j at sample point
            basis_value = evaluate_sh_basis(j, sample_point)
            
            integral_sum += basis_value * material_value * sample_point.weight
            
        coeffs[j] = integral_sum
        
    return coeffs

def compute_light_sh_coefficients(light_sources, max_order):
    """
    Computes SH coefficients for the incident radiance Li.
    """
    coeffs = [0.0] * ((max_order + 1) ** 2)
    
    # Project the combined incident light from all sources onto SH basis
    for j in range((max_order + 1) ** 2):
        integral_sum = 0.0
        for sample_point in sphere_quadrature_points():
            # Evaluate incident radiance at sample point
            # Li(x, omega_i) is the sum of contributions from all light sources
            light_value = evaluate_incident_radiance(sample_point, light_sources)
            
            # Evaluate basis function Y_j at sample point
            basis_value = evaluate_sh_basis(j, sample_point)
            
            integral_sum += basis_value * light_value * sample_point.weight
            
        coeffs[j] = integral_sum
        
    return coeffs

def evaluate_material_function(direction):
    """
    Evaluates the material function M for a given direction.
    This is a placeholder for the specific material model (reflectance/scattering).
    """
    # The material function is defined over the whole sphere.
    # It captures scattering effects.
    return 1.0 # Placeholder

def evaluate_incident_radiance(direction, light_sources):
    """
    Evaluates the incident radiance Li from light sources for a given direction.
    """
    total_radiance = 0.0
    for light in light_sources:
        # Check if light is visible in this direction
        # For area lights, this involves integration over the light surface
        # For point/directional lights, it's a delta function or constant
        if is_light_visible(light, direction):
            total_radiance += light.intensity * light.color
    return total_radiance

def evaluate_sh_basis(index, direction):
    """
    Evaluates the spherical harmonic basis function Y_j for a given direction.
    """
    # Convert index to (l, m)
    l = int((1 + (1 + 8 * index)**0.5) / 2) - 1
    m = index - l * (l + 1)
    
    # Get spherical coordinates (theta, phi) from direction
    theta, phi = direction_to_spherical(direction)
    
    # Evaluate Y_lm(theta, phi)
    # Y_lm uses associated Legendre polynomials for longitude (theta)
    # and complex exponentials for latitude (phi)
    return spherical_harmonic_value(l, m, theta, phi)

def sphere_quadrature_points():
    """
    Returns a set of points on the sphere with weights for numerical integration.
    """
    # Use a standard quadrature scheme (e.g., HEALPix or Gauss-Legendre)
    # This is a placeholder for the actual quadrature implementation
    return []

def direction_to_spherical(direction):
    """
    Converts a 3D direction vector to spherical coordinates (theta, phi).
    """
    # theta: polar angle (0 to pi)
    # phi: azimuthal angle (0 to 2pi)
    theta = acos(direction.z / norm(direction))
    phi = atan2(direction.y, direction.x)
    return theta, phi

def spherical_harmonic_value(l, m, theta, phi):
    """
    Computes the value of the spherical harmonic Y_lm at (theta, phi).
    """
    # Y_lm(theta, phi) = N_lm * P_lm(cos(theta)) * exp(i * m * phi)
    # N_lm is a normalization constant
    # P_lm is the associated Legendre polynomial
    
    # This is a simplified implementation. 
    # In practice, use a robust library or precomputed tables.
    # For real-valued SH, use real spherical harmonics.
    
    # Placeholder for actual SH evaluation
    return 0.0 # Placeholder

def is_light_visible(light, direction):
    """
    Checks if a light source is visible in the given direction.
    """
    # For directional lights, check if direction matches light direction
    # For area lights, check if direction intersects the light surface
    return True # Placeholder

def get_emission(volume_data):
    """
    Retrieves emission values from the volume data.
    """
    # Emission is typically stored in the volume data or a separate texture
    return 0.0 # Placeholder

def perform_dvr_compositing(volume_data, radiance):
    """
    Performs standard Direct Volume Rendering compositing.
    """
    # Standard front-to-back or back-to-front compositing
    # using the computed radiance and volume opacity/density
    return [0.0] # Placeholder
```

## Operating characteristics
*   **Time Complexity:** The core integration step is reduced to a dot product of SH coefficients, which is $O((L+1)^2)$. With $L=3$, this is a constant 16 multiplications and additions per voxel/light interaction [S1].
*   **Memory Complexity:** Requires storing $(L+1)^2$ coefficients per material/light. For $L=3$, this is 16 floats per item [S1].
*   **Performance:** Implemented on the GPU to achieve interactive frame rates [S1]. Allows interactive changes to light and material properties [S1].
*   **Strengths:** Supports colored area light sources and arbitrary light directions. Handles scattering over the whole sphere, which is critical for area light sources with multiple spatial components [S1].
*   **Limitations:** It is a heuristic approach. The approximation of complex material functions with a limited SH order (L=3) may lose high-frequency details [S1]. The lack of surface normal weighting may produce physically inaccurate results for surface-like materials, but is consistent with the volume data definition [S1].

## Background
*   **Origin:** Proposed by Florian Lindemann and Timo Ropinski from the Visualization and Computer Graphics Research Group (VisCG), Department of Computer Science, University of Münster, Germany [S1].
*   **Paper:** "Advanced Light Material Interaction for Direct Volume Rendering" (2010) [S1].
*   **History:** Previous techniques by Schott et al. and Patel et al. used slice-wise front-to-back compositing and only supported headlights [S1]. Techniques by Kniss et al. and Ropinski et al. assumed scattering was directed along one major axis (cone-shaped phase functions), which is a strong approximation, especially for area light sources [S1]. This method extends previous work by using SH basis functions to unify material and lighting representation, allowing for more realistic environmental effects [S1].

## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
