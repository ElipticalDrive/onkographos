# Sky View LUT with Angular Compression

Also called: Seb Hillaire’s Sky and Atmosphere Rendering Technique

## Overview
A technique for rendering sky and atmosphere using a precomputed Look-Up Table (LUT) with angular compression, ported from a reference implementation [S1, unofficial]. It sets itself apart from generic linear parameterizations by using non-linear mappings for altitude and angles to better distribute sampling density according to the physical behavior of light scattering in spherical atmospheres, building on [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Observer position (altitude $h$), view direction (zenith angle $\vartheta$, azimuth), sun direction (zenith angle $\vartheta_s$, azimuth $\phi_s$).
*   **Outputs:** Fragment color $I_F$ for the atmosphere.

**Data Structures:**
*   **Sky View LUT:** A 3D texture (or 2D texture with tiling for the fourth dimension) storing precomputed radiance values. The example implementation uses a resolution of $400 \times 400$ [S1, unofficial].
*   **Atmospheric Parameterization:** A coordinate remapping function that maps the 4D input space $(h, \vartheta, \vartheta_s, \phi_s)$ to 3D texture coordinates $(U, V, W)$ with an emulated fourth dimension $R$ via tiling [9405-atmospheric-parameterization.md].

**Steps:**
1.  **Precomputation:** Numerically integrate single scattering along rays through a spherical atmosphere shell for various observer altitudes and viewing angles. Store the results in the LUT. The extinction coefficient $t$ is precomputed into a 2D lookup texture parameterized by observer altitude $h$ and zenith angle $\vartheta$, containing $t(P P_c)$, where $P$ has altitude $h$ and $P_c$ is the intersection of the ray starting at $P$ under zenith angle $\vartheta$ with the upper atmosphere boundary [S2].
2.  **Runtime Lookup:** For a given pixel, calculate the 4D coordinates based on the observer's position and the sun/view directions.
3.  **Texture Sampling:** Sample the LUT using the mapped 3D coordinates. If the fourth dimension is emulated via tiling, perform manual linear interpolation for the fourth dimension [S2].
4.  **Fragment Color:** The fetched value is the fragment color $I_F = I_S$ for the atmosphere [S2].

**Edge Cases and Pitfalls:**
*   If the observer is outside the atmosphere, compute the intersection point $P_a$ between the view ray and the atmosphere sphere, evaluate the scattering function $f$ for this point, and set scattering/attenuation to zero [S2].
*   Large cell sizes in the LUT can give a blocky look [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sky(observer_altitude, view_zenith, view_azimuth, sun_zenith, sun_azimuth):
    # Map 4D input to 3D texture coordinates with emulated 4th dimension
    u, v, w, r = atmospheric_parameterization(observer_altitude, view_zenith, sun_zenith, sun_azimuth - view_azimuth)
    
    # Check if observer is outside atmosphere
    if observer_altitude > atmosphere_radius:
        # Compute intersection point Pa between view ray and atmosphere sphere
        Pa = compute_intersection(observer_position, view_direction, atmosphere_radius)
        if Pa is None:
            return vec3(0.0)
        # Evaluate scattering function f for Pa
        return evaluate_scattering(Pa, sun_direction)
    
    # Sample LUT
    # Manual linear interpolation for the fourth dimension R if emulated via tiling
    color = lerp(
        sample_lut(u, v, w, floor(r)),
        sample_lut(u, v, w, ceil(r)),
        fract(r)
    )
    
    return color
```

## Subtopics

### [Atmospheric Parameterization](9405-atmospheric-parameterization.md)
Atmospheric Parameterization is the coordinate remapping function used to store precomputed single-scattering data into a 3D texture, enabling efficient runtime lookup for sky and atmosphere rendering. It reduces the four-dimensional input space (altitude, view-zenith angle, sun-zenith angle, sun-view azimuth) into a 3D texture coordinate space (U, V, W) with an emulated fourth dimension (R) via tiling. It sets itself apart from generic linear parameterizations by using non-linear mappings for altitude and angles to better distribute sampling density according to the physical behavior of light scattering in spherical atmospheres, building on [Sky View LUT with Angular Compression](6648-sky-view-lut-with-angular-compression.md).

### Atmosphere Rendering with Lookup Textures
This variant represents the planet and atmospheric shell as finely tessellated spheres with frontface culling (atmosphere) and backface culling (planet). It precomputes the extinction coefficient $t$ into a 2D lookup texture parameterized by observer altitude $h$ and zenith angle $\vartheta$. For rendering, it fetches two values from the scattering texture and performs manual linear interpolation for the fourth dimension. If the observer is outside the atmosphere, it computes the intersection point $P_a$ between the view ray and atmosphere sphere, evaluates $f$ for this point, and sets scattering/attenuation to zero. The fragment color is $I_F = I_S$ for the atmosphere [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for LUT lookup, assuming constant texture size.
*   **Memory Complexity:** $O(N^3)$ for a 3D LUT, where $N$ is the resolution. The example uses a $400 \times 400$ LUT [S1, unofficial].
*   **Strengths:** Efficient real-time rendering of atmospheric scattering. Non-linear parameterization improves visual quality by concentrating samples where scattering changes rapidly.
*   **Limitations:** Precomputation cost is high. Accuracy depends on LUT resolution and parameterization quality. Blocky artifacts can appear if cell sizes are too large [S1, unofficial].
*   **When to use:** Real-time sky and atmosphere rendering in games and simulations.
*   **When not to use:** When high-fidelity spectral scattering is required without precomputation, or when dynamic atmospheric parameters change too frequently for precomputation to be feasible.

## Background
The technique is based on Seb Hillaire’s "A Scalable and Production Ready Sky and Atmosphere Rendering Technique" [S1, unofficial]. It has been ported to various engines, including vxray [S1, unofficial]. The underlying principles of using lookup textures for atmospheric scattering were explored in earlier works such as [Bruneton and Neyret 2008] and [Schafhitzel et al. 2007] [S2].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
[S2] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
