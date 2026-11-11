# Heterogeneous Volumetric Raymarching

Also called: Delta/Ratio Tracking for Participating Media, Volume Raymarching

## Overview
Heterogeneous Volumetric Raymarching is a rendering technique for participating media (such as clouds, fog, and godrays) that utilizes delta tracking and ratio tracking on density fields, rather than sphere tracing on distance functions, to handle non-uniform density distributions [S1]. It distinguishes itself from homogeneous volume methods by explicitly sampling a density field at each step to determine scattering probabilities, and from previous roadmap phases that relied on distance-based sphere tracing, by sharing the step-loop infrastructure while employing different sampling mathematics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray origin and direction; a medium defined by an Axis-Aligned Bounding Box (AABB) and a reference to a density field; light sources (sun, environment map); rendering parameters (step limits, density scale, phase function anisotropy).
*   **Output:** Radiance contribution at the ray origin, including scattered light from the medium and inscatter from lights.

**Data Structures:**
*   **SHAPE_VOLUME:** A new shape variant registered in the Bounding Volume Hierarchy (BVH) leaf nodes. It contains:
    *   `aabb`: The bounding box of the medium.
    *   `density_field_ref`: A reference to the density field data structure.
*   **Density Field:** Accessed via a unified sampler interface supporting:
    *   3D Textures (if RHI supports it).
    *   Procedural generation (e.g., FBM noise + Worley noise) [S1].

**Algorithm Steps:**

1.  **Ray-Medium Intersection:** Determine if the ray intersects the `SHAPE_VOLUME` AABB. If not, return no contribution. If yes, calculate the entry and exit points of the ray within the medium.
2.  **Initialization:** Initialize the current position $p$ to the ray entry point. Initialize accumulated transmittance $T = 1.0$. Initialize path throughput $W = 1.0$. Set bounce counter $b = 0$.
3.  **Marching Loop:** While $b < r\_vol\_multiscatter\_bounces$ and $p$ is within the medium bounds:
    a. **Density Sampling:** Sample the density $\rho(p)$ at the current position $p$ using the unified sampler interface. Apply global density scale: $\rho_{eff} = \rho(p) \times r\_vol\_density\_scale$ [S1].
    b. **Delta Tracking:** Perform delta tracking to find the distance $dt$ to the next potential scattering event. This involves sampling a uniform random number $\xi \in [0, 1)$ and solving for $dt$ such that:
       $$ \exp\left(-\int_{0}^{dt} \sigma_t(p + s\vec{d}) ds\right) = \xi $$
       where $\sigma_t$ is the extinction coefficient, proportional to $\rho_{eff}$ [S1]. In practice, this is often approximated using the maximum density $\rho_{max}$ in the medium for efficiency, leading to ratio tracking.
    c. **Ratio Tracking:** Adjust the probability of scattering based on the ratio of the current density to the maximum density. If a scattering event is selected, proceed to step 4. If not, advance $p$ by $dt$ and repeat step 3a. If $p$ exceeds the medium bounds, terminate the loop [S1].
    d. **Step Size Limiting:** Ensure $dt$ does not exceed $r\_vol\_max\_steps$ remaining steps or a predefined maximum step size to prevent aliasing (see Pitfalls) [S1].
4.  **Scattering Event:**
    a. **Phase Function:** Compute the scattered direction $\vec{\omega}_s$ using the Henyey-Greenstein phase function with anisotropy parameter $g = r\_vol\_phase\_g$. An optional dual-lobe configuration (forward and back-scatter) can be used for clouds [S1].
    b. **Radiance Contribution:** Compute the scattered radiance from the medium itself (if emissive) or from inscatter.
    c. **Next Event Estimation (NEE):** For inscatter from lights (sun, environment map):
       i. Sample a light direction $\vec{\omega}_l$.
       ii. Perform ratio-tracking transmittance calculation back along the ray from $p$ to the light source to determine the probability of light reaching $p$ without being absorbed/scattered [S1].
       iii. Add the weighted radiance contribution to the total.
    d. **Russian Roulette:** With probability $1 - q$, terminate the path. With probability $q$, continue. If continuing, update $W$ by dividing by $q$, increment $b$, and return to step 3 [S1].
5.  **Termination:** If the loop terminates (exit medium or Russian roulette), return the accumulated radiance.

**Parameters:**
*   `r_vol_max_steps`: Maximum number of steps per raymarch. Default: 64 [S1].
*   `r_vol_density_scale`: Global multiplier for medium density. Default: 1.0 [S1].
*   `r_vol_phase_g`: Anisotropy parameter for the Henyey-Greenstein phase function. Default: 0.8 (canonical for clouds) [S1].
*   `r_vol_multiscatter_bounces`: Number of multi-scattering bounces via Russian roulette. Default: 2 [S1].

**Pitfalls:**
*   **Undersampling:** A ray grazing a thin cloud shell at the limb has a long chord length. If the step size ($dt$) is too large relative to the feature size of the density field, it produces aliased noise. Fix: Use cone-footprint stepping ($dt = \text{clamp}(t \cdot \text{pixel\_cone\_angle}, dt\_min, dt\_max)$) and density LOD (drop FBM octaves proportionally to $\log_2(dt / \text{feature\_size})$) [S1].
*   **Performance Overhead:** A new marching loop in the hot path adds compile-time and runtime overhead to all scenes. Fix: Gate the feature behind a `PT_*_ENABLED` macro to exclude it from scenes that do not use volumes [S1].
*   **RHI Limitations:** The engine's RHI may lack 3D texture support (no depth field in `TextureDesc`, no samplers, bilinear interpolation is hand-written). Using 3D textures requires significant new RHI code (~400 LOC) and unproven `RWTexture3D` lowering on Slang targets. Procedural density has no such cost [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of scattering events and steps per ray. The complexity is influenced by the density field evaluation cost and the number of NEE samples.
*   **Memory Complexity:** Depends on the density field representation. 3D textures require memory proportional to their resolution. Procedural generation requires minimal memory.
*   **Measured Performance:** Target performance is $\le 6$ ms at 1080p for a single cloud layer on RTX-class hardware at 1 sample per pixel (spp) [S1].
*   **Strengths:** Handles heterogeneous density distributions effectively; integrates with existing BVH and raymarching infrastructure; supports both procedural and texture-based density fields.
*   **Limitations:** Performance sensitive to step size and density field resolution; potential aliasing if not properly stepped; RHI limitations may hinder 3D texture usage; additional overhead in scenes without volumes if not gated [S1].
*   **When to Use:** Rendering clouds, fog, godrays, and other participating media with non-uniform density.
*   **When Not to Use:** Simple homogeneous volumes (use simpler methods); scenes where performance is critical and volumes are not present (use gating).

## Background
Heterogeneous Volumetric Raymarching was introduced as Phase 4 of the "Roadmap: SDF + raymarching: surface and volumetric integration" (#96) in the demont-engine project [S1]. It was proposed by havokentity on May 16, 2026 [S1]. The method is designed to render cumulus cloud layers, godrays, and non-uniform fog, and is intended to pair with spherical-Earth atmospheric models (#51) for curved aerial perspective [S1]. It is independent of Phases 1-3 of the roadmap, which used sphere tracing on distance, but shares the step-loop pattern and Slang module structure [S1].

## Sources
[S1] renderer: heterogeneous volumetric raymarching -- clouds, godrays, non-uniform fog (Phase 4 of #96), https://github.com/havokentity/demont-engine/issues/100, fetched 2026-10-26
