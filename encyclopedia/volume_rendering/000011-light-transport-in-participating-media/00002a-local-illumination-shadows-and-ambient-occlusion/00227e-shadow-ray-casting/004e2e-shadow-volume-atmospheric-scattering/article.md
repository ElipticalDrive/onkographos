# Shadow Volume Atmospheric Scattering

Also called: Real Time Atmospheric Scattering with Volumetric Shadows, Rendering Algorithm for Participating Media with Shadows

## Overview
A real-time rendering method for atmospheric light scattering and volumetric shadows that computes the direct contribution of a participating medium on any surface using an analytical expression of the light transport equation, avoiding precomputation and texture memory [S1, unofficial]. It differs from volume rendering methods (e.g., shadow maps) by not requiring texture memory and avoiding sampling artifacts for sharp edges like light shafts [S1, unofficial]. It differs from other direct representation methods (e.g., Sun et al. [SR05]) by integrating volumetric shadows [S1, unofficial].

## How it works
**Inputs:**
*   Scene geometry (triangular meshes, 2-manifold with connectivity).
*   Participating media definitions (homogeneous, constant extinction coefficient $k_t$).
*   Light sources (one or several static or moving point lights).
*   Camera position.

**Outputs:**
*   Final image with atmospheric scattering and volumetric shadows.

**Data Structures:**
*   **Shadow Plane Mesh:** A mesh generated from object silhouettes. Vertices store the analytical medium contribution value.
*   **Stencil Buffer:** Used to track lit areas and manage shadow volume logic.
*   **Silhouette Loop List:** Linked list of edges forming the silhouette of shadow casters.

**Parameters:**
*   $k_t$: Extinction coefficient (constant per medium) [S1, unofficial].
*   Light intensity $I_s$ [S1, unofficial].
*   Shadow plane mesh density: Finely meshed near the light, coarsely far away, based on radial distribution [S1, unofficial].
*   Polynomial approximation degree: 4th degree for phase function approximation [S1, unofficial].

**Steps:**

1.  **Compute Silhouettes:**
    For every moving shadow caster, determine silhouettes regarding the light position. Identify all edges common to a front-facing triangle and one back-facing triangle. Link these edges together and store in a loop list [S1, unofficial].

2.  **Render Scene (Surface + Ambient):**
    *   Render the scene classically with light attenuation due to absorption and scattering.
    *   Compute surface luminance $L(P)$ for each point $P$:
        $$ L(P) = \rho_P \frac{I(\omega_P) \cos(\theta_P)}{\pi r_P^2} e^{-k_t r_P} $$
        where $\rho_P$ is surface reflectivity, $I$ is light intensity in direction $\omega$ toward $P$, $\theta_P$ is the angle between surface normal and light direction, and $r_P$ is distance to light [S1, unofficial].
    *   Add ambient fog to account for absorption and multiple scattering.
    *   Compute hard surface shadows using a stencil algorithm.
    *   Disable lighting at the end of this stage. The stencil buffer now contains lit areas (value 1) and shadowed areas (value 0) [S1, unofficial].

3.  **Compute Medium Contribution on Scene Vertices:**
    *   Render the scene once more to add medium contribution for each vertex using additive blending.
    *   Set depth test to equality.
    *   Use the stencil buffer to render only lit parts of the scene.
    *   Compute the medium contribution $L_m(P)$ for each vertex using the analytical expression (Equation 2 below) [S1, unofficial].

4.  **Sort Shadow Planes:**
    *   Sort shadow planes in back-to-front order.
    *   Distance metric: Cosine of the angle between vector $\vec{SO}$ (camera to light) and vector $\vec{SP}$ (camera to silhouette edge center) [S1, unofficial].

5.  **Render Shadow Planes:**
    *   Render shadow planes in the sorted order.
    *   Depth test accepts only fragments closer to the camera.
    *   **Front-facing planes:** Add contribution. Stencil function allows fragments if stencil equals 1. Increment stencil buffer by +1.
    *   **Back-facing planes:** Subtract contribution. Stencil function allows fragments if stencil equals 0. Decrement stencil buffer by -1.
    *   Render simple quads with stencil function set to 'always' to work around hardware limitations regarding stencil increment/decrement [S1, unofficial].
    *   For bounded media, set stencil to 255 when rendering back-facing triangles of the medium boundary to avoid further rendering [S1, unofficial].

**Analytical Formula for Medium Contribution:**
The method uses an angular formulation variable change: $r = \sqrt{h^2 + (x - t)^2}$, allowing integration with respect to angle $\theta$ instead of distance $x$ [S1, unofficial].

The medium contribution $L_m(P)$ is approximated using a 4th-degree polynomial base for the phase function:
$$ p(\theta + \pi/2) \approx c_0(k_t, h) + c_1(k_t, h)\theta + c_2(k_t, h)\theta^2 + c_3(k_t, h)\theta^3 + c_4(k_t, h)\theta^4 $$
where coefficients $c_i$ depend on extinction coefficient $k_t$ and height $h$ [S1, unofficial].

The final analytical solution for a non-directional point light source is:
$$ L_m(P) = \frac{k_t e^{-k_t t} \Omega}{4\pi h} \left[ c_0(k_t, h)[\theta]_{\theta_0}^{\theta_d} + c_1(k_t, h)\left[\frac{\theta^2}{2}\right]_{\theta_0}^{\theta_d} + \dots \right] $$
This is computed in constant time without numerical integration [S1, unofficial].

For a ray passing through shadowed regions, the contribution is decomposed into a sum of differences between lit segments defined by shadow planes and medium boundaries:
$$ L_m(P) = (\Gamma_m(F) - \Gamma_m(E)) + (\Gamma_m(D) - \Gamma_m(C)) + (\Gamma_m(B) - \Gamma_m(A)) $$
where $\Gamma_m(P)$ is the expression (2) for a lit ray between camera center $O$ and point $P$, points $A, F$ are on the medium boundary, and $B, C, D, E$ are on shadow planes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_medium_contribution_analytical(theta_0, theta_d, k_t, h, omega, t):
    """
    Computes Lm(P) using the analytical polynomial approximation.
    theta_0, theta_d: Integration limits in radians.
    k_t: Extinction coefficient.
    h: Height parameter from angular formulation.
    omega: Solid angle factor.
    t: Distance parameter from angular formulation.
    Returns: Medium contribution Lm(P).
    """
    # Coefficients c0 to c4 depend on k_t and h.
    # In a real implementation, these would be precomputed or looked up.
    # For this pseudocode, we assume a function get_coefficients exists.
    c = get_coefficients(k_t, h)
    
    # Polynomial integration terms
    # Integral of theta^n is theta^(n+1) / (n+1)
    # Limits are theta_0 to theta_d
    
    term_0 = c[0] * (theta_d - theta_0)
    term_1 = c[1] * (theta_d**2 / 2 - theta_0**2 / 2)
    term_2 = c[2] * (theta_d**3 / 3 - theta_0**3 / 3)
    term_3 = c[3] * (theta_d**4 / 4 - theta_0**4 / 4)
    term_4 = c[4] * (theta_d**5 / 5 - theta_0**5 / 5)
    
    sum_terms = term_0 + term_1 + term_2 + term_3 + term_4
    
    prefactor = (k_t * math.exp(-k_t * t) * omega) / (4 * math.pi * h)
    
    return prefactor * sum_terms

def render_shadow_planes(shadow_planes, camera_pos, light_pos, stencil_buffer):
    """
    Renders shadow planes in back-to-front order, updating stencil and framebuffer.
    """
    # Sort planes back-to-front using cosine of angle between SO and SP
    sorted_planes = sort_back_to_front(shadow_planes, camera_pos, light_pos)
    
    for plane in sorted_planes:
        if plane.is_front_facing:
            # Add contribution
            set_stencil_func(equal, 1) # Allow if stencil is 1
            set_stencil_op(increment) # Increment stencil
            render_plane_additive(plane)
        else:
            # Subtract contribution
            set_stencil_func(equal, 0) # Allow if stencil is 0
            set_stencil_op(decrement) # Decrement stencil
            render_plane_subtractive(plane)
            
        # Handle bounded media boundary back-facing triangles
        if plane.is_medium_boundary_back_facing:
            set_stencil_func(always)
            set_stencil_op(replace, 255)
            render_plane(plane)

def get_coefficients(k_t, h):
    """
    Returns coefficients c0 to c4 for the polynomial approximation.
    Exact formulas depend on the phase function (isotropic, hazy, etc.).
    Not given in the sources. [G]
    """
    # Placeholder for actual coefficient calculation based on [S1, unofficial]
    # Annex of the source paper contains formal expressions.
    return [0.0, 0.0, 0.0, 0.0, 0.0]

def sort_back_to_front(planes, camera_pos, light_pos):
    """
    Sorts planes based on cosine of angle between SO and SP.
    S = light_pos, O = camera_pos, P = silhouette edge center.
    """
    def sort_key(plane):
        # Vector SO
        vec_so = light_pos - camera_pos
        # Vector SP (using edge center)
        vec_sp = plane.edge_center - camera_pos
        
        # Cosine similarity
        cos_val = dot(vec_so, vec_sp) / (length(vec_so) * length(vec_sp))
        return cos_val
    
    # Sort descending to render back-to-front (higher cosine = further back in this metric?)
    # The source says "back to front" and uses cosine. 
    # Figure 5 implies ordering based on this metric.
    return sorted(planes, key=sort_key, reverse=True)
```

**Edge Cases and Pitfalls:**
*   **Shadow-in-shadows:** Occurs if shadow planes are not rendered in back-to-front order or if stencil is not used to skip shadowed planes. Prevented by sorting planes and using stencil to reject shadowed shadow planes [S1, unofficial].
*   **Framebuffer Clamping:** Causes darkening if medium contribution exceeds 1 and is later subtracted. Avoided by choosing reasonable light intensities or using float textures (though blending in float buffers was not supported at the time) [S1, unofficial].
*   **Approximation Errors:** Occur when the ray passes close to the source (contribution is so high) or when the observer is far from the source (contribution is so small); these errors remain unnoticeable [S1, unofficial].
*   **Bounded Media:** Coefficients $r$ and $x$ in exponentials must be distances to the medium boundary. The method approximates $r$ to the average distance $R$ between a point on the boundary and a point in the medium [S1, unofficial].

## Subtopics

### Real-Time Single Scattering Algorithm with Exact Shadow Planes
A real-time algorithm computing single scattering for one or several participating media, utilizing exact shadow planes to avoid sampling aliasing artifacts for both surface and volumetric shadows [S1, unofficial]. It computes participating medium contributions in software, orders and computes shadow planes in software, and renders shadow plane contributions, which represent the main computational cost [S1, unofficial]. It is an improvement over other atmospheric scattering algorithms, especially when a medium covers the whole scene, achieving >25 FPS for moderately complex scenes [S1, unofficial]. It handles moving lights, changing colors, and complex scenes with ~100,000 triangles [S1, unofficial]. Pitfalls include avoiding darkness by choosing a reasonable light source intensity or using a float texture [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Constant time per ray for the scattering contribution calculation (no numerical integration) [S1, unofficial].
*   **Measured Performance:** 120 FPS at 800x600 resolution with 32 shadow planes on a 2.6 GHz processor and ATI 9800 graphic card [S1, unofficial]. Comparison: Dobashi's algorithm achieved 12.5 FPS at 450x300 resolution for the same test scene, mainly due to texture rendering accumulation and high fill rate [S1, unofficial].
*   **Workload Distribution:**
    *   Fig 7 left: Stage 1 (1%), Stage 2 (8.7%), Stage 3 (21.7%), Stage 4 (9.6%), Stage 5 (59%) [S1, unofficial].
    *   Fig 7 right: Stage 1 (1.7%), Stage 2 (14%), Stage 3 (33.6%), Stage 4 (16%), Stage 5 (34.7%) [S1, unofficial].
*   **Strengths:** Real-time performance, no texture memory cost, no sampling aliasing artifacts for sharp edges like light shafts, supports moving lights and objects [S1, unofficial].
*   **Limitations:** Single scattering only (multiple scattering approximated by constant ambient term), homogeneous medium assumption, framebuffer clamping issues if not managed, requires 2-manifold meshes with connectivity for silhouette computation [S1, unofficial].
*   **When to use:** Real-time rendering of atmospheric effects and volumetric shadows in scenes where texture memory is limited or sharp light shafts are required.
*   **When not to use:** Scenes requiring multiple scattering, inhomogeneous media, or soft shadows (without further extension) [S1, unofficial].

## Background
Originated in the paper "Real Time Rendering of Atmospheric Scattering and Volumetric Shadows" [S1, unofficial]. It was developed to address the difficulty of rendering volumetric shadows in real-time, providing a great performance improvement over previous methods like Dobashi et al. [DY02] [S1, unofficial]. The method is close to shadow volume techniques [He91] and uses shadow planes [AA03] [S1, unofficial]. It is used in the paper's proposed algorithm for real-time rendering of atmospheric effects and volumetric shadows [S1, unofficial].

## Sources
[S1] Real Time Rendering of Atmospheric Scattering and Volumetric Shadows, https://hal.science/hal-00681568v1, fetched 2026-10-26 (unofficial)
