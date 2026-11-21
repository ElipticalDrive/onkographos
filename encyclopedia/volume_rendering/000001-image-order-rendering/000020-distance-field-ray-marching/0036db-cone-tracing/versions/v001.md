# Cone Tracing

Also called: Cone Marching, Cone Tracing for Soft Shadows

## Overview
Cone tracing is an extension of sphere tracing used in Signed Distance Field (SDF) rendering to approximate soft shadows and occlusion by expanding a cone along the marching direction [S1, unofficial]. It differs from standard sphere tracing by accumulating opacity based on the cone's radius relative to the SDF, allowing for soft shadow estimation with minimal additional computational cost compared to hard shadow ray marching [S1, S3, unofficial].

## How it works
**Inputs:**
*   Surface point $x$ and light direction $\mathbf{l}$ [S3, unofficial].
*   Signed Distance Field (SDF) function $S(\mathbf{p})$ [S1, unofficial].
*   Cone aperture angle $\theta$ (controls softness) [S3, unofficial].
*   Initial step size $d$ (often derived from SDF value) [S1, unofficial].

**Outputs:**
*   Shadow intensity or occlusion value $\alpha$ [S2, S3, unofficial].

**Data Structures:**
*   No complex structures required; relies on iterative sampling of the SDF [S1, unofficial].

**Steps:**
1.  Initialize current position $\mathbf{p} = x$ and accumulated occlusion $\alpha = 0$ [S3, unofficial].
2.  Initialize cone radius $r = 0$ [S1, unofficial].
3.  Loop while $\mathbf{p}$ is within bounds and $\alpha < 1$:
    a.  Query SDF: $s = S(\mathbf{p})$ [S1, unofficial].
    b.  If $s < 0$, the point is inside geometry; break and return fully occluded [G].
    c.  Compare $s$ with current cone radius $r$. If $s < r$, the cone has intersected an object [S1, unofficial].
    d.  Calculate local occlusion contribution $\alpha_2$ based on the ratio $s/r$ or similar metric [S3, unofficial].
    e.  Update accumulated occlusion: $\alpha = \alpha + (1-\alpha)\alpha_2$ [S3, unofficial].
    f.  Advance position: $\mathbf{p} = \mathbf{p} + s \cdot \mathbf{l}$ (or fixed step) [S1, unofficial].
    g.  Expand cone radius: $r = r + \Delta r$, where $\Delta r$ is determined by the aperture angle $\theta$ and step size [S3, unofficial].
4.  Return shadow intensity $1 - \alpha$ [S2, unofficial].

**Formulas:**
*   Cone Radius Expansion: The radius $r$ increases linearly with distance. For an aperture angle $\theta$, the radius at distance $d$ from the origin is $r = d \cdot \tan(\theta/2)$ [S3, unofficial].
*   Occlusion Update: $\delta = \delta + (1-\delta)\alpha_2 f(r)$, where $f(r)$ is a decay factor [S3, unofficial].

**Parameters:**
*   Aperture angles tested: 1°, 5°, 10°, 25° [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Geometric Interpretation:** The statement "If the SDF value is smaller than the current radius, the cone has intersected an object" is inaccurate if interpreted as a strict geometric cone [S1, unofficial]. The shape traced by linearly increasing radius is a cone with a hemisphere on the end [S1, unofficial]. Intersection occurs if the distance to the nearest surface is less than the current test radius at any point along the horizontal center line [S1, unofficial].
*   **Artifacts:** Implementing cone tracing for soft shadows is tricky and prone to artifacts or light leaking [S2, unofficial].
*   **Cost:** It is computationally expensive [S2, unofficial].

## Subtopics

### Soft Shadows via Cone Tracing
Soft shadows are achieved by tracing a cone from the surface point $x$ towards the direction of the light [S3, unofficial]. The cone aperture controls how soft and scattered the resulting shadow is [S3, unofficial]. Only the occlusion value $\alpha_2$ is accumulated at each step [S3, unofficial]. Aperture values tested include 1, 5, 10, and 25 degrees [S3, unofficial].

## Operating characteristics
*   **Complexity:** Linear in the number of steps; each step requires one SDF query and simple arithmetic [S1, unofficial].
*   **Performance:** Described as incurring "very little additional cost" to normal sphere trace in some contexts [S1, unofficial], but also described as "quite expensive" and "tricky to get shadows without artifacts" in other implementations [S2, unofficial].
*   **Strengths:** Provides soft shadow approximation with minimal overhead over hard shadows [S1, unofficial].
*   **Limitations:** Prone to light leaking and artifacts; expensive for high-quality soft shadows [S2, unofficial].

## Background
Cone tracing is an extension of sphere tracing used for SDF rendering [S1, unofficial]. It was discussed in the context of voxel cone tracing for real-time global illumination [S3, unofficial] and in a thesis on cone tracing [S1, unofficial].

## Sources
[S1] Reading thesis on cone tracing, did the author make a mistake?, https://computergraphics.stackexchange.com/questions/9508/reading-thesis-on-cone-tracing-did-the-author-make-a-mistake, fetched 2026-10-26 (unofficial)
[S2] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S3] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
