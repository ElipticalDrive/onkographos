# Hybrid Ray Tracing

Also called: Hybrid Ray Tracer, Hybrid Ray Tracing for Geometric/Volumetric Mixing, Mixing Polygons and Volumes

## Overview
Hybrid Ray Tracing is a method for mixing geometric and volumetric data by casting rays through the ensemble of both, drawing samples of each, and compositing them in depth-sorted order [S1, unofficial][S2, unofficial]. It solves the problem of rendering scenes containing both polygonal objects (e.g., surgical instruments, skin surfaces) and volumetric data (e.g., CT/MR scans) simultaneously. It sets itself apart from pure volume rendering [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) by explicitly handling polygon intersections and from pure ray tracing [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md) by integrating volumetric attenuation and emission along the ray path.

## How it works
**Inputs:**
1.  A 3D voxel array of scalar values (density/opacity) and associated color/opacity transfer functions [S2, unofficial].
2.  A set of embedded polygons (boundary representation) [S2, unofficial].
3.  Camera position and viewing direction [S2, unofficial].

**Outputs:**
*   Final image color for each pixel [S2, unofficial].

**Data Structures:**
*   **Volumetric Data:** $N \times N \times N$ array of scalar values. Each voxel is classified to yield a color $c_v$ and opacity $o_v$ [S2, unofficial].
*   **Geometric Data:** List of polygons with vertices and normals.
*   **Ray State:** Current position, accumulated color $C$, accumulated opacity $O_{acc}$, and intermediate buffers for supersampling checks ($C_x, C_q$) [S2, unofficial].

**Numbered Steps:**

1.  **Volumetric Classification:** Shade and classify the volumetric data to yield a color $c_v$ and opacity $o_v$ for each voxel [S2, unofficial].
2.  **Ray Casting:** Trace parallel viewing rays from the observer position into the data [S2, unofficial].
3.  **Volumetric Sampling:** For each ray, compute a vector of colors $c_v$ and opacities $o_v$ by resampling the volumetric data at equally spaced positions along the ray. Use tri-linear interpolation from the colors and opacities in the eight voxels surrounding each sample location [S2, unofficial].
    *   The spacing between samples along the ray equals the slab thickness [S2, unofficial].
    *   The spacing between adjacent rays equals the slab width [S2, unofficial].
4.  **Geometric Intersection:** Independently compute all intersections between the ray and polygons in the environment. Shade them to yield a color $c_p$ and opacity $o_p$ for each point of intersection [S2, unofficial].
5.  **Slab Decomposition (Polygon Handling):** When a ray intersects a polygon within a slab, treat the polygon locally as a plane perpendicular to the ray at the intersection point [S2, unofficial].
    *   Calculate the opacity of the volumetric sliver immediately in front of the plane ($o_f$) and behind the plane ($o_b$) using exponential attenuation formulas [S2, unofficial].
    *   Let $o_v$ be the opacity of the entire slab, $t_v$ be the total slab thickness, $t_f$ be the thickness of the front portion, and $t_b$ be the thickness of the back portion [S2, unofficial].
    *   Compute $o_f$ and $o_b$ using the formulas below.
6.  **Depth-Sorted Compositing:** Composite resampled volumetric colors and opacities with polygonal colors and opacities in depth-sorted order [S2, unofficial].
    *   If a ray intersects more than one polygon within a slab, compute and composite the contribution of each polygon and each sliver of volumetric gel separately [S2, unofficial].
    *   Working from front to back, composite the front volumetric sliver ($c_v, o_f$), then the polygon ($c_p, o_p$), then the back volumetric sliver ($c_v, o_b$) [S2, unofficial].
7.  **Adaptive Supersampling (Aliasing Avoidance):** To avoid aliasing at polygonal edges, intersections, and shading highlights, increase the number of rays cast per pixel in these regions [S2, unofficial].
    *   Compute two additional colors during ray tracing to determine if supersampling is needed [S2, unofficial]:
        *   $C_x$: Contributions only from polygons (set $o_v=0$ for all volumetric samples).
        *   $C_q$: Contributions only from polygons, attenuated by passage through volumetric data (set $o_v=0$ for all volumetric samples, but apply volumetric attenuation to the polygon contribution) [S2, unofficial].
    *   Let $\theta$ be a threshold [S2, unofficial].
    *   If range($C_x$) > $\theta$ and range($C_q$) $\le \theta$: Color difference is due to volumetric data. Do not supersample [S2, unofficial].
    *   If range($C_q$) > $\theta$ and range($C_x$) $\le \theta$: Geometric event is hidden by opaque volumetric data. Do not supersample [S2, unofficial].
    *   If range($C_x$) > $\theta$ and range($C_q$) > $\theta$: Region contains a visible geometric event. Cast more rays (supersample) [S2, unofficial].

**Formulas:**
*   **Slab Opacity:**
    $$o_v = 1 - e^{-d_v t_v}$$
    Where $d_v$ is density, $t_v$ is thickness, and $0 < o_v < 1$ [S2, unofficial].
*   **Front Sliver Opacity:**
    $$o_f = 1 - (1 - o_v)^{t_f/t_v}$$
    Where $t_f$ is the thickness of the front portion [S2, unofficial].
*   **Back Sliver Opacity:**
    $$o_b = 1 - (1 - o_v)^{t_b/t_v}$$
    Where $t_b$ is the thickness of the back portion [S2, unofficial].

**Pitfalls:**
*   Volumetric samples lying immediately in front of and behind polygons require special treatment to avoid errors in visibility [S1, unofficial].
*   Distinguishing whether color differences are due to volumetric or geometric data is critical for correct supersampling decisions [S2, unofficial].

## Subtopics

### Hybrid Ray Tracing with Volume Data
A rendering method that combines shaded polygons with volumetric data by treating surfaces as infinitesimally thin solids, using a binary voxel representation for the geometry, and rendering the composite dataset using volume rendering techniques to avoid aliasing artifacts [S1, unofficial]. Unlike the primary hybrid ray tracer which uses adaptive supersampling, this approach converts polygons to a voxel grid. It requires bandlimiting the object to the Nyquist frequency in all three dimensions prior to sampling to avoid aliasing artifacts [S1, unofficial]. If polygons are sufficiently bandlimited, no supersampling is required [S1, unofficial].

## Operating characteristics
*   **Complexity:** The method involves ray-polygon intersection tests and volumetric sampling for every ray. Adaptive supersampling increases the cost in regions with geometric complexity.
*   **Performance:** Exact hidden-volume solutions inside every slab are extraordinarily expensive [S2, unofficial]. The approximation of treating polygons as local planes perpendicular to the ray provides a satisfactory balance of accuracy and expense [S2, unofficial].
*   **Strengths:** Handles mixtures of geometric and volumetric data correctly; avoids aliasing of polygonal edges via supersampling or bandlimiting [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires careful handling of visibility near polygon boundaries; supersampling adds computational cost [S2, unofficial].

## Background
The idea of a hybrid ray tracer that handles both scalar fields and polygons has been proposed many times, but implementations were rare at the time of writing [S1, unofficial]. Levoy [1988a] described volume rendering algorithms. The specific hybrid method extending Levoy's algorithm to handle polygonally defined objects was presented by Levoy [1988b] (cited as [S2, unofficial]). Kaufman et al. [1990] also discussed mixing polygons and volumes, noting that converting polygonal models into sampled volumes is a straightforward way to achieve this [S3, unofficial].

## Sources
[S1] DTIC ADA208091: Interactive Visualization of 3D Medical Data, https://archive.org/details/DTIC_ADA208091, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S3] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
