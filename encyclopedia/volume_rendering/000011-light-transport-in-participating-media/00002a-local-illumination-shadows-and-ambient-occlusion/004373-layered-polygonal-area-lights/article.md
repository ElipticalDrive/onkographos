# Layered Polygonal Area Lights

Also called: Layered Polygonal Area Light (LPAL) Integration, Volume slicing with area lights

## Overview
Layered Polygonal Area Lights (LPAL) is a real-time rendering technique for computing the indirect illumination of emissive, inhomogeneous participating media as reflected off glossy surfaces [S1]. It solves the problem of efficiently evaluating the volume integral for light transport from a volume to a surface point by decomposing the volume into parallel planar slices, each treated as a virtual area light [S1]. Unlike Voxel Cone Tracing (VCT) or sparse VPL sampling, LPAL analytically integrates light contributions over these cutting sections while accounting for extinction within the intervals between slices, providing higher visual quality for glossy reflections at comparable computational costs [S1].

## How it works
The method approximates the total radiance reflected from a point $\mathbf{x}_i$ on a reflective surface by integrating contributions from all vertices $\mathbf{v}$ within the volume domain $V$ [S1].

**Inputs and Outputs:**
*   **Input:** 3D texture representing the volume domain $V$ (typically an Axis-Aligned Bounding Box, AABB); surface point $\mathbf{x}_i$ with normal $\mathbf{n}(\mathbf{x}_i)$ and BRDF $f_r$; camera center $\mathbf{x}_e$; volume gravity center $\mathbf{c}_V$.
*   **Output:** Radiance $L(\mathbf{x}_i, \omega_o)$ reflected towards the camera direction $\omega_o = (\mathbf{x}_e - \mathbf{x}_i) / \|\mathbf{x}_e - \mathbf{x}_i\|$ [S1].

**Data Structures:**
*   **Volume Domain $V$:** Defined by an AABB. If multiple disjoint volumes exist, they should be stored in individual bounding boxes to avoid excessive slicing [S1].
*   **LPAL Layers:** A set of parallel planes perpendicular to a cutting direction $\omega_c$. The number of layers is a parameter; more layers retain detail for sparse or distant volumes [S1].

**Cutting Direction:**
The direction $\omega_c$ defines the cutting sections. In practice, it is chosen as the line connecting the reflection point $\mathbf{x}_i$ and the gravity center of the domain $V$ [S1]. Let $s$ be the signed distance from $\mathbf{x}_i$ along $\omega_c$. Let $A(s)$ be the plane intersecting $\omega_c$ perpendicularly at distance $s$ [S1].

**Formulas:**
The total radiance is the integral over the volume $V$:
$$ L(\mathbf{x}_i, \omega_o) = \int_V L(\mathbf{v} \to \mathbf{x}_i \to \mathbf{x}_e) \, d\mathbf{v} $$
where the radiance from a single vertex $\mathbf{v}$ is:
$$ L(\mathbf{v} \to \mathbf{x}_i \to \mathbf{x}_e) = \frac{I(\mathbf{v}, -\omega_i)}{\|\mathbf{v} - \mathbf{x}_i\|^2} \text{Tr}(\mathbf{v} \to \mathbf{x}_i) f_r(\mathbf{x}_i; \omega_i, \omega_o) |\omega_i \cdot \mathbf{n}(\mathbf{x}_i)| $$
Here, $\omega_i = (\mathbf{v} - \mathbf{x}_i) / \|\mathbf{v} - \mathbf{x}_i\|$ is the incident direction, $I$ is radiant intensity, and $\text{Tr}$ is transmittance [S1].

The method decomposes this into an integral over slices $s$:
$$ L(\mathbf{x}_i, \omega_o) \approx \int L(s) \, ds $$
where $L(s)$ is the contribution from plane $A(s)$ [S1]:
$$ L(s) = \int_{A(s)} L(\omega_i) f_r(\mathbf{x}_i; \omega_i, \omega_o) |\omega_i \cdot \mathbf{n}(\mathbf{x}_i)| \, d\omega_i $$
This integral is separated into a surface term $ID$ and a light term $IL$:
$$ L(s) \approx ID \cdot IL $$
$$ ID = \int_{A(s)} f_r(\mathbf{x}_i; \omega_i, \omega_o) |\omega_i \cdot \mathbf{n}(\mathbf{x}_i)| \, d\omega_i $$
$$ IL = \frac{\int_{A(s)} L(\omega_i) f_r(\mathbf{x}_i; \omega_i, \omega_o) |\omega_i \cdot \mathbf{n}(\mathbf{x}_i)| \, d\omega_i}{\int_{A(s)} f_r(\mathbf{x}_i; \omega_i, \omega_o) |\omega_i \cdot \mathbf{n}(\mathbf{x}_i)| \, d\omega_i} $$
The term $ID$ is computed analytically using Linearly Transformed Cosine (LTC) integration in clamped cosine space [S1]. The term $IL$ is approximated by filtering the area light with Gaussian functions and storing the result in a MIP map [S1].

The transmittance $\text{Tr}(\mathbf{v} \to \mathbf{x}_i)$ is approximated using an average extinction coefficient $\bar{\sigma}_t$ over the slice:
$$ \text{Tr}(\mathbf{v} \to \mathbf{x}_i) \approx e^{-\bar{\sigma}_t \|\mathbf{v}_{//}(s) - \mathbf{x}_i\|} $$
where $\mathbf{v}_{//}(s) = \mathbf{x}_i + s\omega_c$ is the vertex on the reflected ray at distance $s$ [S1].

**Algorithm Steps:**
1.  **Define Volume Domain:** Identify the AABB enclosing the emissive volume $V$. If multiple volumes are present, use separate bounding boxes [S1].
2.  **Determine Cutting Direction:** Calculate $\omega_c$ as the vector from $\mathbf{x}_i$ to the gravity center of $V$ [S1].
3.  **Slice Volume:** Divide the volume along $\omega_c$ into $N$ parallel planes (LPALs) at distances $s_k$ [S1].
4.  **Compute Average Extinction:** For each slice $k$, calculate the weighted average extinction coefficient $\bar{\sigma}_t(s_k)$. Simple averaging is insufficient; the method weights the integration based on the BRDF lobe shape, considering properties of the volume near the regularly reflected ray [S1].
5.  **Evaluate Light Term ($IL$):** For each slice, approximate the light integral $IL$ by filtering the area light contribution with Gaussian functions, utilizing a pre-computed MIP map [S1].
6.  **Evaluate Surface Term ($ID$):** Compute the BRDF integral $ID$ analytically using LTC in clamped cosine space [S1].
7.  **Accumulate Radiance:** Sum the contributions $ID \cdot IL$ across all slices, applying the transmittance attenuation $e^{-\bar{\sigma}_t \Delta s}$ between slices [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_lpal_radiance(x_i, n, omega_o, volume_texture, volume_center, camera_center, num_slices):
    """
    Computes indirect illumination from an emissive volume using LPAL.
    """
    # 1. Setup geometry
    omega_i_reflected = reflect(omega_o, n) # Direction towards volume for reflection
    # Note: The paper uses omega_c from x_i to volume center.
    # In the context of the integral, we integrate over the volume.
    # The paper defines omega_c as the line connecting x_i and gravity center.
    omega_c = normalize(volume_center - x_i)
    
    # Define the range of s (signed distance) covering the volume
    # Project volume bounds onto omega_c to find min_s and max_s
    min_s, max_s = get_volume_depth_range(volume_texture, x_i, omega_c)
    
    slice_step = (max_s - min_s) / num_slices
    total_radiance = 0.0
    
    # Pre-compute LTC parameters for the surface BRDF
    ltc_params = compute_ltc_parameters(n, omega_o)
    
    for k in range(num_slices):
        s = min_s + k * slice_step
        s_next = s + slice_step
        
        # 2. Get slice properties
        # Average extinction coefficient weighted by BRDF lobe
        sigma_t_avg = get_weighted_average_extinction(
            volume_texture, x_i, omega_c, s, slice_step, ltc_params
        )
        
        # 3. Compute Transmittance from slice to surface
        # Distance from surface to the center of the slice plane
        dist = s
        transmittance = exp(-sigma_t_avg * dist)
        
        # 4. Compute Light Term (IL)
        # Approximated by filtering area light with Gaussian functions
        # Stored in MIP map based on slice distance and size
        light_term = get_filtered_light_term(
            volume_texture, x_i, omega_c, s, slice_step, ltc_params
        )
        
        # 5. Compute Surface Term (ID)
        # Analytic integration of BRDF over the hemisphere/slice
        surface_term = compute_ltc_integral(ltc_params)
        
        # 6. Accumulate
        # Contribution of this slice
        slice_radiance = surface_term * light_term * transmittance
        total_radiance += slice_radiance * slice_step
        
    return total_radiance
```

**Edge Cases and Pitfalls:**
*   **Black Margins:** Standard 3D textures have black (zero) margins. If the volume data is in the center of the AABB, these margins cause insufficiently blurred reflections on highly rough surfaces because the LTC integration considers the zero-emission margin [S1]. The method requires handling or eliminating black margins, e.g., by filling the domain with uniform emittance or disabling domain transforms that introduce margins [S1].
*   **High Roughness:** High roughness values cause deviations from path tracing due to the black margin issue in the LTC-based surface integration [S1].
*   **Simple Averaging:** Using a simple average for extinction coefficients fails to account for spatial variance of density and albedo, inappropriately attenuating light from the sides when the BRDF has a sharp lobe [S1].
*   **Multiple Volumes:** If multiple volumes are stored in a single large bounding box, they require more slices to retain detail. Storing them in individual bounding boxes is recommended [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of slices $N$. The per-slice cost involves texture lookups and LTC evaluation, which is constant time.
*   **Memory Complexity:** Requires storage for the volume texture and potentially a MIP map for the filtered light terms.
*   **Performance:** Designed for real-time rendering. It is significantly faster than path tracing and provides better visual quality than VPL sampling with limited samples (e.g., 64 VPLs) [S1].
*   **Strengths:** Accurately calculates area integration over cutting sections; considers light extinction within intervals; applicable to surfaces with arbitrary shapes and roughness [S1].
*   **Limitations:** Relies on observation-based approximations rather than pure physics; sensitive to black margins in 3D textures; less accurate for very high roughness compared to path tracing [S1].
*   **When to Use:** Real-time rendering of participating media indirectly visible over reflective surfaces, especially for glossy materials [S1].
*   **When Not to Use:** When strict physical accuracy is required for highly rough surfaces, or when the volume is extremely sparse and requires a very high number of slices for detail, potentially negating performance benefits.

## Background
The Layered Polygonal Area Lights (LPAL) method was proposed by T. Kuge et al. in 2019 [S1]. It was introduced as a real-time technique for indirect illumination of emissive inhomogeneous volumes. The method builds upon the Linearly Transformed Cosine (LTC) model for surface integration [HDHN16]. It addresses the limitations of Voxel Cone Tracing (VCT) and VPL sampling in the context of glossy reflections from participating media [S1].

## Sources
[S1] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
