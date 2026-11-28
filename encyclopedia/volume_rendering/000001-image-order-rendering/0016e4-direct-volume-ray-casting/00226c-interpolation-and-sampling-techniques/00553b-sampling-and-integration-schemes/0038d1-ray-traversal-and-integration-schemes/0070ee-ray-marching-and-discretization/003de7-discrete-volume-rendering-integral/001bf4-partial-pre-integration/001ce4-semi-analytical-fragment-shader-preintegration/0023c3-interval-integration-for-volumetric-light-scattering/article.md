# Interval Integration for Volumetric Light Scattering

Also called: Analytic Interval Integration, Sum & Difference of Intervals

## Overview
Interval Integration computes in-scattered radiance by decomposing the integration path into intervals defined by the front and back faces of a light volume, allowing analytical evaluation for directional lights and numerical or lookup-based evaluation for point and spot lights [S1, unofficial]. It sets itself apart from its parent [Semi-Analytical Fragment Shader Preintegration](1379-semi-analytical-fragment-shader-preintegration.md) by explicitly using the geometric boundaries of the light volume (front/back faces) to define integration limits, rather than relying solely on local extrema of scalar polynomials [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin, ray direction, light volume geometry (front and back faces), light type (directional, omnidirectional, spot), shadow map, and medium properties (density, phase function).
*   **Outputs:** In-scattered radiance $L_S$ for the pixel.

**Data Structures:**
*   **Light Volume Mesh:** Defined by front faces ($G_f$) and back faces ($G_b$).
*   **Lookup Texture:** For omnidirectional lights, a 2D texture mapping $\cos^{-1}(v \cdot l)$ to precomputed differential values [S1, unofficial].
*   **Shadow Map:** Depth buffer used to determine visibility of ray segments [S1, unofficial].

**Steps:**

1.  **Decompose Ray into Intervals:**
    Identify the intersection points of the view ray with the light volume. The ray is divided into segments based on entering ($n \in G_f$) and exiting ($m \in G_b$) the volume [S1, unofficial].

2.  **Evaluate In-Scattered Radiance per Light Type:**
    The total in-scattered radiance $L_S$ is computed as the sum of contributions from lit intervals [S1, unofficial]:
    $$ L_S = \sum_{n \in G_f} L'_S(t_n) - \sum_{m \in G_b} L'_S(t_m) $$
    where $L'_S(t)$ is the indefinite integral of the scattering term evaluated at distance $t$ [S1, unofficial].

    *   **Directional Lights:**
        Assume constant direction and constant power (light distance >> medium depth). The integral simplifies to an analytic function evaluated in the pixel shader [S1, unofficial]:
        $$ L'_S(d) = L(d, l) \rho_m(x_l, \omega_x) \frac{1 - e^{-\tau_{ex} d}}{\tau_{ex}} $$
        where $d$ is the distance along the ray, $L$ is light intensity, $\rho_m$ is medium reflectance, and $\tau_{ex}$ is extinction coefficient [S1, unofficial].

    *   **Omnidirectional Lights:**
        Analytical solutions are too complex. Create a lookup texture mapping $\cos^{-1}(v \cdot l)$ to 2D space. Evaluate the differential and numerically integrate using a Compute Shader [S1, unofficial].

    *   **Spot Lights:**
        Perform cone intersection to clamp the interval. For variable falloff, perform numerical integration using Newton-Cotes methods in the pixel shader [S1, unofficial]. For fixed or no falloff, use modified point light lookups or treat as point lights [S1, unofficial].

3.  **Apply Attenuation:**
    Ensure that the light reaching the camera is attenuated by the medium [S2, unofficial].

4.  **Composite Results:**
    Additively blend the resolved/filtered output with the scene. Use dual-source blending to attenuate based on medium and depth [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_interval_integration(ray_origin, ray_direction, light_volume, light_type, shadow_map):
    # 1. Find intersections with light volume
    intersections = find_ray_mesh_intersections(ray_origin, ray_direction, light_volume)
    
    # Sort intersections by distance
    intersections.sort(key=lambda p: p.distance)
    
    # 2. Decompose into intervals
    intervals = []
    for i in range(0, len(intersections) - 1, 2):
        if i + 1 < len(intersections):
            entry = intersections[i]
            exit = intersections[i + 1]
            # Check if interval is lit using shadow map
            if is_lit(entry, exit, shadow_map):
                intervals.append((entry.distance, exit.distance))
    
    # 3. Sum contributions
    total_radiance = 0.0
    
    for start_dist, end_dist in intervals:
        if light_type == "directional":
            # Analytic solution for directional light
            # L'_S(d) = L(d, l) * rho_m * (1 - exp(-tau_ex * d)) / tau_ex
            term_start = analytic_directional_integral(start_dist)
            term_end = analytic_directional_integral(end_dist)
            contribution = term_end - term_start
        elif light_type == "omnidirectional":
            # Lookup texture + numerical integration
            # Map cos^-1(v . l) to 2D space
            cos_angle = dot(normalize(ray_direction), normalize(light_direction))
            lookup_val = lookup_texture(cos_angle)
            # Numerical integration using Compute Shader results or precomputed table
            contribution = numerical_integral_omni(start_dist, end_dist, lookup_val)
        elif light_type == "spot":
            # Cone intersection and Newton-Cotes
            cone_intersection = check_cone_intersection(start_dist, end_dist, light_direction, spot_angle)
            if cone_intersection:
                contribution = newton_cotes_integration(start_dist, end_dist, light_falloff)
            else:
                contribution = 0.0
        else:
            contribution = 0.0
            
        # Apply attenuation
        attenuation = exp(-extinction_coefficient * (end_dist - start_dist))
        total_radiance += contribution * attenuation
        
    return total_radiance

def analytic_directional_integral(d):
    # Simplified analytic solution
    # L'_S(d) = L * rho_m * (1 - exp(-tau_ex * d)) / tau_ex
    # Assuming constant L, rho_m, tau_ex for the interval
    return light_intensity * medium_reflectance * (1.0 - exp(-extinction_coefficient * d)) / extinction_coefficient

def is_lit(entry, exit, shadow_map):
    # Check if the segment between entry and exit is occluded
    # Sample shadow map at entry and exit points
    shadow_entry = sample_shadow_map(entry)
    shadow_exit = sample_shadow_map(exit)
    # If both are lit (or based on specific shadow map logic), return True
    return shadow_entry and shadow_exit
```

**Edge Cases and Pitfalls:**
*   **Shadow Map Inconsistencies:** If only front faces are rendered to the shadow map, or if distant occluders are excluded, artifacts appear. Ensure shadow map contents are consistent with scene geometry (e.g., render both front and back faces if needed) [S1, unofficial].
*   **Temporal AA Jitter:** Temporal AA jitter causes flickering in down-sampled buffers. Add a separate TAA resolve to down-sampled buffers before compositing [S1, unofficial].
*   **High-Frequency Occluders:** Performance drops at low angles due to dense occluders. Adjust tessellation factor based on view angle and pre-filter shadow maps [S1, unofficial].

## Subtopics

### Semi-analytical Inscattering for Point Lights
A method to compute the inscattering integral for a point light source by reducing the integral to a function of three variables ($h$, $S_c$, $S_o$) and using a 2D precomputed lookup table, avoiding the lack of a closed-form solution [S2, unofficial]. It precomputes the inscattering integral for different values of $h$ (distance from light to ray) and start distance $S_c$ (signed distance from light projected position to camera) to infinity, storing results in a two-dimensional lookup table [S2, unofficial]. The inscattering integral for a specific ray segment is computed as the integral from the camera towards infinity minus the integral from the object towards infinity, with attenuation applied to the light reaching the camera in the media [S2, unofficial]. The phase function $p(\theta)$ is assumed to be normalized to unity such that $\frac{1}{4\pi} \frac{dp(\theta)}{dw} = 1$ where integration is performed over the whole set of directions $w$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of intervals (intersections with light volume) and the complexity of the light type. Directional lights are $O(1)$ per interval (analytic). Omnidirectional and spot lights require lookup or numerical integration, which is more expensive [S1, unofficial].
*   **Memory Complexity:** Requires storage for the light volume mesh, shadow maps, and lookup textures for omnidirectional/spot lights [S1, unofficial].
*   **Performance:** Fast enough for entire spec range and flexible enough for almost any engine [S1, unofficial]. Compatible with physically-based engines and available in DirectX 11 [S1, unofficial].
*   **Strengths:** Analytic solution for directional lights is efficient. Handles complex light types via lookups and numerical integration.
*   **Limitations:** Shadow map inconsistencies can cause artifacts. Temporal jitter requires additional resolve steps. High-frequency occluders can cause performance drops [S1, unofficial].
*   **When to Use:** Real-time volumetric light scattering in games and interactive applications, particularly when directional lighting is dominant [S1, unofficial].

## Background
Originated from work on volumetric light scattering in real-time rendering, specifically integrated into the game *Fallout 4* [S1, unofficial]. The method builds on earlier analytic single scattering models [S1, unofficial]. It was developed to provide a fast, flexible, and physically-based solution for volumetric lighting in modern game engines [S1, unofficial].

## Sources
[S1] Volumetric Light Scattering in Fallout 4, https://www.scribd.com/document/748135006/Fast-Flexible-Physically-Based-Volumetric-Light-Scattering, fetched 2026-10-29 (unofficial)
[S2] High-Performance Light Scattering Rendering, https://www.scribd.com/document/748135031/822368Yusov-Egor-Practical-Implementation-of-Light, fetched 2026-10-29 (unofficial)
