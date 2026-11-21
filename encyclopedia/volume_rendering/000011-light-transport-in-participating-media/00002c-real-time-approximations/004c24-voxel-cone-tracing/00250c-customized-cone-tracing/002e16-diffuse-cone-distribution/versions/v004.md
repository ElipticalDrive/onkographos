# Diffuse Cone Distribution

Also called: 5-Cone Diffuse Sampling, Normal-Perpendicular Cone Distribution

## Overview
Diffuse Cone Distribution is a specific sampling pattern used in Voxel Cone Tracing to approximate indirect diffuse lighting in participating media or surface-based global illumination [S1, unofficial]. It solves the problem of efficiently estimating hemispherical irradiance using a minimal number of cone traces. It is distinguished from its siblings by its rigid geometric symmetry: one cone aligned with the surface normal and four cones distributed in the perpendicular plane, as opposed to the random rotations of [Random Cone Rotation](5911-random-cone-rotation.md) or the multi-bounce logic of [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md).

## How it works
**Inputs:** Surface position $P$, surface normal $n$, voxel grid data.
**Outputs:** Accumulated diffuse indirect lighting color $L_{diffuse}$.
**Parameters:**
- Number of cones $N = 5$ [S1, unofficial].
- Cone aperture $\theta = 30^\circ$ [S1, unofficial].

**Data Structures:**
The algorithm maintains a list of 5 cone directions. Each cone is defined by its axis direction vector $d_i$ and the fixed aperture $\theta$.

**Steps:**
1.  **Construct Basis Vectors:**
    Let $n$ be the normalized surface normal. Construct two orthogonal vectors $u$ and $v$ such that $\{u, v, n\}$ form an orthonormal basis. This can be done by finding a vector $t$ not parallel to $n$, setting $u = \text{normalize}(t - (t \cdot n)n)$, and $v = n \times u$ [G].

2.  **Define Cone Directions:**
    Generate the 5 direction vectors $d_i$ for $i \in \{0, \dots, 4\}$:
    -   $d_0 = n$ (The primary cone along the normal).
    -   $d_1 = u$
    -   $d_2 = -u$
    -   $d_3 = v$
    -   $d_4 = -v$
    These four vectors ($d_1$ to $d_4$) lie in the plane perpendicular to $n$ [S1, unofficial].

3.  **Trace Cones:**
    For each cone direction $d_i$:
    a.  **Offset Origin:** To prevent self-intersection artifacts (self-shadowing/self-lighting), offset the ray origin $P$ slightly along the normal: $P' = P + \epsilon n$, where $\epsilon$ is a small epsilon value [S1, unofficial].
    b.  **Cone Trace:** Perform a voxel cone trace starting at $P'$ along direction $d_i$ with aperture $\theta$. This involves stepping through the voxel grid, accumulating transmittance and in-scattering (or reflected color) within the cone volume. Let the result be $C_i$.

4.  **Accumulate:**
    Sum the results:
    $$ L_{diffuse} = \sum_{i=0}^{4} C_i $$
    Note: The sources do not specify weighting factors for the perpendicular cones versus the normal cone. In standard implementations, equal weighting is often assumed unless normalized by the solid angle, but the source text implies a direct sum [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Self-Shadowing:** Failing to offset the ray origin causes the cone to immediately intersect the surface voxel, resulting in incorrect darkening or brightening [S1, unofficial].
-   **Blocky Artifacts:** With only 5 cones, the sampling pattern is sparse and geometrically rigid, leading to visible blockiness in the indirect lighting [S1, unofficial]. This is the primary motivation for the sibling method [Random Cone Rotation](5911-random-cone-rotation.md).

## Subtopics

### [Random Cone Rotation](5911-random-cone-rotation.md)
Mitigates the blocky artifacts of this fixed distribution by randomly rotating the cone set around the normal. Choose this when visual quality is prioritized over deterministic performance, accepting the introduction of noise.

## Operating characteristics
-   **Complexity:** $O(1)$ cone traces per surface point (fixed at 5).
-   **Performance:** Very fast due to the low cone count. However, the fixed aperture of $30^\circ$ [S1, unofficial] may be too narrow for highly rough surfaces or too wide for smooth ones, potentially missing grazing angle contributions or including too much noise.
-   **Limitations:** The rigid 5-cone structure produces noticeable aliasing/blocking artifacts in indirect lighting [S1, unofficial]. It does not adapt to surface roughness for diffuse components (unlike the specular cones described in the same source).

## Background
This distribution originates from practical implementations of Voxel Cone Tracing for real-time rendering, specifically detailed in a GitHub report by righier [S1, unofficial]. It represents a minimal viable solution for approximating hemispherical diffuse integration in voxel-based global illumination systems.

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
