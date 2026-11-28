# Analytical Integration of Scattering Phase Function for Random Polygons

Also called: Calculating the scattering phase function, Projected area and diffuse/specular integration for random polygons

## Overview
This method computes the contribution of a random collection of small polygons (total area 1) to the radiative transfer equation by analytically integrating projected area, diffuse reflection, and specular terms over angular bins of surface normals [S1, book]. It neglects mutual occlusion, assuming the probability of random polygon overlap approaches zero in an infinitesimal slab [S1, book]. It sets itself apart from numerical Monte Carlo approaches by providing closed-form or precomputed analytical solutions for the integration of geometric terms over solid angle bins, enabling efficient evaluation of the scattering phase function for vegetation-like media.

## How it works

### Inputs and Outputs
**Inputs:**
*   Viewing direction $\mathbf{V}$ (unit vector).
*   Light direction $\mathbf{L}$ (unit vector).
*   Surface normal distribution accumulated into angular bins $S_i = [\theta_i, \theta_{i+1}]$ on the unit sphere.
*   Material coefficients: diffuse reflectivity $k_d$, specular exponent $c$, and optionally separate coefficients per wavelength band for reflection and transmission [S1, book].
*   Polygon type: one-sided (opaque solid) or two-sided (translucent leaf) [S1, book].

**Outputs:**
*   Integrated radiance contribution for each bin, normalized by the solid angle of the bin.

### Data Structures
*   **Angular Bins:** A set of intervals $S_i$ covering the range of polar angles $\theta_N$ for surface normals.
*   **Precomputed Table:** Values for the function $Q(t)$ used in specular integration.

### Step-by-Step Method

1.  **Define Coordinate System and Normal/View Geometry:**
    Express the surface normal $\mathbf{N}$ and viewing direction $\mathbf{V}$ in spherical coordinates $(\theta_N, \phi_N)$ and $(\theta_V, \phi_V)$ respectively:
    $$ \mathbf{N} = (\sin \theta_N \cos \phi_N, \sin \theta_N \sin \phi_N, \cos \theta_N) $$
    $$ \mathbf{V} = (\sin \theta_V \cos \phi_V, \sin \theta_V \sin \phi_V, \cos \theta_V) $$
    The dot product is:
    $$ \mathbf{N} \cdot \mathbf{V} = \sin \theta_N \sin \theta_V (\cos \phi_N \cos \phi_V + \sin \phi_N \sin \phi_V) + \cos \theta_N \cos \theta_V $$

2.  **Determine Visible Range $R$:**
    Find the range of $\phi_N$ where $\mathbf{N} \cdot \mathbf{V} > 0$. Solve $\mathbf{N} \cdot \mathbf{V} = 0$ for $\phi_N$. Define $\Delta \phi = |\phi_N - \phi_V|$:
    $$ \Delta \phi = (\cos^{-1})(-\cot \theta_N \cot \theta_V) $$
    Where the function $(\cos^{-1})(x)$ is defined as:
    $$ (\cos^{-1})(x) = \begin{cases} \cos^{-1}(x) & \text{if } -1 < x < 1 \\ 0 & \text{if } x > 1 \\ \pi & \text{if } x < -1 \end{cases} $$
    The visible range is $R = [\phi_V - \Delta \phi, \phi_V + \Delta \phi]$ (modulo $2\pi$) [S1, book].

3.  **Compute Projected Area $\Delta A$:**
    *   **One-sided polygons:** Integrate $\mathbf{N} \cdot \mathbf{V}$ over $R$. The total projected area $\Delta A_{one-sided}$ is:
        $$ \Delta A_{one-sided} = \frac{1}{\pi} \sin \theta_N \sin \theta_V \sin \Delta \phi + \frac{1}{\pi} \Delta \phi \cos \theta_N \cos \theta_V $$
        [Note: The source text shows $\Delta \phi \cos \theta_N \cos \theta_V$ term, but the derivation implies integration of the constant part. The final formula given in the source for two-sided is used below to derive one-sided consistency, but we stick to the explicit one-sided result if available. The source gives the two-sided result explicitly. For one-sided, it is the integral over $R$. The source states: "So for one-sided polygons, the projected area $\Delta A$ is... [formula cut off in OCR but implied by the two-sided subtraction/addition logic]". However, the source explicitly provides the two-sided formula. Let us use the explicit two-sided formula and the logic for one-sided.]
        
        Actually, the source provides the explicit result for two-sided:
        $$ \Delta A_{two-sided} = \frac{1}{\pi} \sin \theta_N \sin \theta_V \sin \Delta \phi + \frac{1}{\pi} (2\pi - \Delta \phi) \cos \theta_N \cos \theta_V $$
        For one-sided, the integral is only over $R$. The source implies the one-sided integral is the first term plus the integral of the constant part over $R$. The constant part of $\mathbf{N} \cdot \mathbf{V}$ is $\cos \theta_N \cos \theta_V$. The range $R$ has width $2\Delta \phi$. Thus:
        $$ \Delta A_{one-sided} = \frac{1}{\pi} \sin \theta_N \sin \theta_V \sin \Delta \phi + \frac{1}{\pi} \Delta \phi \cos \theta_N \cos \theta_V $$
        [G: Derived from the structure of the two-sided formula and the definition of $R$. The source text for one-sided is truncated but the two-sided formula adds the integral of $-\mathbf{N} \cdot \mathbf{V}$ over the complement. The complement has width $2\pi - 2\Delta \phi$. The term $\cos \theta_N \cos \theta_V$ integrates to $(2\pi - 2\Delta \phi) \cos \theta_N \cos \theta_V / \pi$? No, the source says "add the integral of $-\mathbf{N} \cdot \mathbf{V}$ ... to get the total". The total is $\Delta A_{two-sided}$. The one-sided is just the integral over $R$. The source explicitly writes the two-sided result. We will use the two-sided result for two-sided cases and the logical complement for one-sided.]

    *   **Two-sided polygons:** Add the integral of $-\mathbf{N} \cdot \mathbf{V}$ over the complementary range $[0, 2\pi] - R$. The total is $\Delta A_{two-sided}$ as defined above [S1, book].

4.  **Compute Diffuse Reflection/Transmission:**
    *   **One-sided Opaque:** Integrate $(\mathbf{N} \cdot \mathbf{L})$ weighted by projected area $\mathbf{N} \cdot \mathbf{V}$ over the region $S$ where $\mathbf{N} \cdot \mathbf{L} > 0$. Region $S = [\phi_L - \Delta \phi_L, \phi_L + \Delta \phi_L]$ where $\Delta \phi_L = (\cos^{-1})(-\cot \theta_N \cot \theta_L)$. Divide by $\Delta A$ (the weight integral). The product $(\mathbf{N} \cdot \mathbf{V})(\mathbf{N} \cdot \mathbf{L})$ is integrated over the intersection of $R$ and $S$ [S1, book].
    *   **Two-sided Translucent:** Divide the unit circle for $\phi_N$ into four ranges based on the overlap of $R$ (visible) and $S$ (lit):
        1.  Front visible, front lit.
        2.  Back visible, front lit.
        3.  Front visible, back lit.
        4.  Back visible, back lit.
        Integrate $|(\mathbf{N} \cdot \mathbf{V})(\mathbf{N} \cdot \mathbf{L})|$ over each range. Ranges (1) and (2) contribute to diffuse reflection; (3) and (4) contribute to diffuse transmission [S1, book].

5.  **Integrate Over Angular Bins:**
    For each bin $S_i = [\theta_i, \theta_{i+1}]$, integrate the computed $K$ (projected area) and diffuse terms using Simpson's rule with respect to $\theta_N$. Divide the result by the solid angle of the band $S_i$ [S1, book].

6.  **Compute Specular Reflection:**
    *   **Blinn-Phong Term:** The term $(\mathbf{V} \cdot \mathbf{H})^c$ is constant on small circles centered at the half-vector $\mathbf{H} = (\mathbf{V}+\mathbf{L})/|\mathbf{V}+\mathbf{L}|$. Flatten the unit sphere near $\mathbf{H}$ onto the tangent plane $T_H$ at $\mathbf{H}$ using map $p_H$, which maps great circle arclength distance to $r = \sqrt{x^2+y^2}$.
    *   Precompute and tabulate:
        $$ Q(t) = \int_0^t \frac{\cos^c(\phi) \sin(\phi)}{\phi} d\phi $$
        [Note: The source writes $\cos^c(\phi) \sin(\phi) / \phi$. The factor $\sin(r)/r$ converts differential area on the plane to sphere. The integral $Q(t)$ handles the angular part.]
    *   Find the integral over the band $S_i$ (approximated as between two parallel lines in the $(x,y)$ plane) using additions/subtractions of $Q(t)$.
    *   Multiply by $\mathbf{V} \cdot \mathbf{H}$ to account for foreshortening (assuming $\mathbf{N} \approx \mathbf{H}$ for significant highlights).
    *   Divide by the solid angle of the band $S_i$ [S1, book].
    *   **Phong Term:** For the Phong term $(\mathbf{V} \cdot \mathbf{R})^c$ where $\mathbf{R} = 2(\mathbf{N} \cdot \mathbf{L})\mathbf{N} - \mathbf{L}$, the method requires integrating over the map $\phi$ that takes $\mathbf{N}$ to $\mathbf{R}$. The source indicates this is handled similarly but does not provide the explicit closed-form $Q(t)$ equivalent for Phong, stating "For the Phong term... we need to integrate...". [S1, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def acos_clipped(x):
    """
    Implements the (cos^-1) function defined in the source.
    """
    if x > 1:
        return 0.0
    elif x < -1:
        return math.pi
    else:
        return math.acos(x)

def compute_delta_phi(theta_n, theta_v):
    """
    Computes Delta phi for a given theta_n and theta_v.
    """
    cot_n = 1.0 / math.tan(theta_n) if math.sin(theta_n) != 0 else 0.0
    cot_v = 1.0 / math.tan(theta_v) if math.sin(theta_v) != 0 else 0.0
    val = -cot_n * cot_v
    return acos_clipped(val)

def integrate_projected_area_one_sided(theta_n, theta_v, phi_v):
    """
    Computes projected area for one-sided polygons.
    """
    delta_phi = compute_delta_phi(theta_n, theta_v)
    term1 = (1.0 / math.pi) * math.sin(theta_n) * math.sin(theta_v) * math.sin(delta_phi)
    term2 = (1.0 / math.pi) * delta_phi * math.cos(theta_n) * math.cos(theta_v)
    return term1 + term2

def integrate_projected_area_two_sided(theta_n, theta_v):
    """
    Computes projected area for two-sided polygons.
    """
    delta_phi = compute_delta_phi(theta_n, theta_v)
    term1 = (1.0 / math.pi) * math.sin(theta_n) * math.sin(theta_v) * math.sin(delta_phi)
    term2 = (1.0 / math.pi) * (2.0 * math.pi - delta_phi) * math.cos(theta_n) * math.cos(theta_v)
    return term1 + term2

def integrate_diffuse_weighted(theta_n, theta_v, phi_v, theta_l, phi_l, is_two_sided):
    """
    Computes the weighted integral for diffuse reflection/transmission.
    Returns (numerator, denominator).
    """
    # Determine R (visible range)
    delta_phi_v = compute_delta_phi(theta_n, theta_v)
    r_start = phi_v - delta_phi_v
    r_end = phi_v + delta_phi_v
    
    # Determine S (lit range)
    delta_phi_l = compute_delta_phi(theta_n, theta_l)
    s_start = phi_l - delta_phi_l
    s_end = phi_l + delta_phi_l
    
    # Intersection of R and S
    # Note: R and S are intervals on the circle [0, 2pi]. 
    # For simplicity, assume no wrapping for this pseudocode structure, 
    # or handle modular arithmetic. The source says "straightforwardly integrated over the intersection".
    
    # Calculate intersection interval [max(starts), min(ends)]
    inter_start = max(r_start, s_start)
    inter_end = min(r_end, s_end)
    
    if inter_start >= inter_end:
        return 0.0, 0.0 # No overlap or degenerate
        
    # The product (N.V)(N.L) is a polynomial in sin(phi_n) and cos(phi_n).
    # The source states it can be "straightforwardly integrated".
    # We approximate this by evaluating the integrand at the midpoint or using a simple rule 
    # if the exact polynomial coefficients are not provided in the notes.
    # However, the notes say "straightforwardly integrated", implying a closed form exists.
    # Since the exact polynomial expansion is not provided in the notes, we mark this as a gap.
    # [G] The exact polynomial coefficients for (N.V)(N.L) in terms of sin/cos(phi_n) are not explicitly listed.
    # Implementer must expand (N.V)(N.L) and integrate term-by-term over [inter_start, inter_end].
    
    # Placeholder for the actual analytical integration of the polynomial
    # numerator = analytical_integral_polynomial(theta_n, theta_v, theta_l, phi_v, phi_l, inter_start, inter_end)
    numerator = 0.0 # Placeholder
    
    # Denominator is the projected area
    if is_two_sided:
        denominator = integrate_projected_area_two_sided(theta_n, theta_v)
    else:
        denominator = integrate_projected_area_one_sided(theta_n, theta_v)
        
    return numerator, denominator

def integrate_specular_blinn(theta_n, theta_v, theta_l, c, solid_angle_band):
    """
    Computes specular contribution using Blinn-Phong model and Q(t) table.
    """
    # Compute Half Vector H
    hx = math.sin(theta_v)*math.cos(phi_v) + math.sin(theta_l)*math.cos(phi_l)
    hy = math.sin(theta_v)*math.sin(phi_v) + math.sin(theta_l)*math.sin(phi_l)
    hz = math.cos(theta_v) + math.cos(theta_l)
    h_len = math.sqrt(hx**2 + hy**2 + hz**2)
    if h_len == 0:
        return 0.0
    hx, hy, hz = hx/h_len, hy/h_len, hz/h_len
    
    # Map to tangent plane at H
    # This requires rotating the coordinate system so H is the Z-axis.
    # Then projecting N onto the XY plane.
    # The distance r is the great circle distance from H.
    # r = acos(N . H)
    
    # The integral is over the band S_i.
    # The source says: "find the integral over the band between any two parallel lines in the (x,y) plane"
    # using Q(t).
    
    # Precomputed Q(t) table lookup would happen here.
    # Q(t) = integral_0^t (cos^c(phi) * sin(phi) / phi) dphi
    
    # Multiply by V.H for foreshortening
    v_dot_h = math.sin(theta_v)*math.cos(phi_v)*hx + math.sin(theta_v)*math.sin(phi_v)*hy + math.cos(theta_v)*hz
    
    # The result is scaled by V.H and divided by solid angle.
    # The exact integration over the band using Q(t) additions/subtractions is complex to pseudocode 
    # without the specific geometry of the band boundaries in the tangent plane.
    # [G] The specific mapping of the band boundaries S_i to the (x,y) tangent plane coordinates 
    # and the subsequent addition/subtraction logic for Q(t) is not fully detailed in the notes.
    
    return 0.0 # Placeholder for Q(t) based integration

def process_bin(theta_i, theta_i_plus_1, phi_v, theta_v, phi_l, theta_l, c, is_two_sided, solid_angle_band):
    """
    Processes a single angular bin S_i.
    """
    # Use Simpson's rule for integration over theta_n from theta_i to theta_i_plus_1
    # Midpoint
    theta_mid = (theta_i + theta_i_plus_1) / 2.0
    
    # Evaluate at endpoints and midpoint
    # Note: Simpson's rule requires 3 points.
    points = [theta_i, theta_mid, theta_i_plus_1]
    weights = [1, 4, 1]
    
    sum_k = 0.0
    sum_diffuse = 0.0
    
    for theta_n, w in zip(points, weights):
        # Projected Area K
        if is_two_sided:
            k = integrate_projected_area_two_sided(theta_n, theta_v)
        else:
            k = integrate_projected_area_one_sided(theta_n, theta_v)
        sum_k += w * k
        
        # Diffuse
        num, den = integrate_diffuse_weighted(theta_n, theta_v, phi_v, theta_l, phi_l, is_two_sided)
        if den > 1e-9:
            diffuse_val = num / den
        else:
            diffuse_val = 0.0
        sum_diffuse += w * diffuse_val
        
    # Simpson's rule factor
    h = (theta_i_plus_1 - theta_i) / 2.0
    
    avg_k = (sum_k * h / 3.0) / solid_angle_band
    avg_diffuse = (sum_diffuse * h / 3.0) / solid_angle_band
    
    # Specular (simplified)
    avg_specular = integrate_specular_blinn(theta_mid, theta_v, theta_l, c, solid_angle_band) / solid_angle_band
    
    return avg_k, avg_diffuse, avg_specular
```

### Parameters
*   **Diffuse Coefficients:** Separate coefficients for diffuse reflection and transmission are allowed per wavelength band [S1, book].
*   **Optical Properties:** Front and back surfaces are assumed to have identical optical properties in the current implementation [S1, book].

### Edge Cases and Pitfalls
*   **Degenerate Angles:** When $\theta_N$ or $\theta_V$ is 0 or $\pi$, $\cot \theta$ is undefined. The $(\cos^{-1})$ function handles values outside $[-1, 1]$ by clamping to 0 or $\pi$, but care must be taken in computing $\cot$ [S1, book].
*   **Empty Intersections:** The intersection of ranges $R$ and $S$ may be empty, resulting in zero contribution for diffuse terms [S1, book].
*   **Precision:** The integration of the polynomial $(\mathbf{N} \cdot \mathbf{V})(\mathbf{N} \cdot \mathbf{L})$ requires careful handling of trigonometric identities to avoid precision loss [S1, book].

## Operating characteristics

*   **Time Complexity:** The method is $O(1)$ per bin per light/view pair if the precomputed $Q(t)$ table is used for specular terms, and $O(1)$ for diffuse terms if the polynomial integration is pre-derived. The overall cost is proportional to the number of angular bins.
*   **Memory Complexity:** Requires storage for the angular bin distribution and the precomputed $Q(t)$ table.
*   **Strengths:** Provides exact analytical integration for geometric terms, avoiding noise from Monte Carlo methods. Efficient for static scenes with known normal distributions.
*   **Limitations:** Neglects mutual occlusion, which may be inaccurate for dense media. Assumes random polygon distribution. The specular integration relies on approximations (flattening to tangent plane) [S1, book].
*   **When to Use:** Suitable for global illumination in vegetation where normal distributions are known or can be accumulated, and real-time or efficient offline rendering is required.
*   **When Not to Use:** For scenes with significant self-shadowing or non-random normal distributions.

## Background

This method originates from the work on plane parallel radiance transport for global illumination in vegetation [S1, book]. It was developed to efficiently compute light scattering in participating media, specifically modeling the interaction of light with leaf-like polygons. The approach leverages the analytical properties of spherical geometry to integrate the scattering phase function over angular bins, enabling the solution of the integro-differential radiative transfer equation for such media.

## Sources

[S1] Plane parallel radiance transport for global illumination in vegetation, https://www.osti.gov/biblio/649229, fetched 2026-10-26 (book)
