# Billboards and particles
Also called: Billboards, Particles, classical billboards, cross billboards

## Overview
Billboards and particles are a method for approximating participating media effects or complex geometry (such as forests) using 2D semi-transparent textures that always face the camera [S2, unofficial][S1]. This approach is primarily chosen for its low computational cost, making it suitable for real-time rendering in industrial simulators and video games where performance is critical [S1][S2, unofficial]. Unlike higher-fidelity volume rendering techniques, it sacrifices geometric accuracy and parallax richness for speed, relying on pre-defined textures or rule-based particle systems [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of billboard instances (static or dynamic), each defined by a position, a 2D semi-transparent texture, and potentially orientation constraints. For particles, inputs also include motion rules, lifecycle parameters (appearance/disappearance), and density integration data if applicable [S2, unofficial].
*   **Outputs:** A 2D image where billboards are composited using alpha blending to approximate the visual effect of the underlying medium or object [S2, unofficial].

**Data Structures:**
*   **Billboard Instance:**
    *   `position`: 3D vector $(x, y, z)$ representing the center or anchor point in world space.
    *   `texture`: Reference to a 2D image resource containing color and alpha channels.
    *   `orientation`: Optional constraint (e.g., axis-aligned vertical) [S1].
*   **Particle Instance:**
    *   Inherits billboard fields.
    *   `velocity`: 3D vector for movement.
    *   `lifetime`: Scalar for age tracking.
    *   `rules`: Logic defining spawn, movement, and death conditions [S2, unofficial].

**Steps:**
1.  **Placement:** Define billboard positions in the scene. For forests, positions are often sparse to mitigate popping artifacts [S1]. For media, positions may be static or derived from simulation data [S2, unofficial].
2.  **Orientation:** For each billboard, compute the view matrix or rotation such that the billboard's normal vector aligns with the camera's view direction (billboarding) [S1][S2, unofficial].
    *   *Classical Billboards:* The quad faces the camera directly. An axis (e.g., vertical) may be constrained to remain upright [S1].
    *   *Cross Billboards:* Generate 2 to 4 quads per instance, rotated around the vertical axis (e.g., 0°, 90°, 180°, 270°) to provide multi-directional coverage [S1].
3.  **Rendering:** Render the quads as textured primitives.
    *   Use alpha blending to composite the semi-transparent textures [S2, unofficial].
    *   For participating media, blend color according to integrated density to improve visual results [S2, unofficial].
4.  **Particle Update (if applicable):** Update particle positions based on velocity and time. Handle appearance and disappearance according to predefined rules [S2, unofficial].

**Formulas:**
*   **Billboard Rotation:** The rotation matrix $R$ for a billboard at position $P$ with camera position $C$ is derived such that the billboard's local Z-axis aligns with the vector $C - P$ [S1][S2, unofficial].
*   **Cross Billboard Coverage:** For $N$ quads (typically $N=2$ to $4$), the $i$-th quad is rotated by angle $\theta_i = \frac{2\pi i}{N}$ around the vertical axis [S1].

**Parameters:**
*   **Number of Quads (Cross Billboards):** 2 to 4 [S1].
*   **Texture Type:** Semi-transparent 2D texture [S2, unofficial].
*   **Placement Density:** Sparse for forests to avoid popping [S1].

**Edge Cases and Pitfalls:**
*   **Parallax Artifacts:** Classical billboards show no parallax when the camera moves. This is only acceptable for axis-symmetrical objects or grazing view angles [S1].
*   **Popping:** In dense scenes (e.g., forests), a tree slightly behind another may "pop" in front due to small camera motions. This requires sparse placement or careful LOD management [S1].
*   **Grazing Angle Artifacts:** Cross billboards exhibit visible artifacts when one of the quads is viewed at a grazing angle. Fading non-facing billboards can mitigate this but introduces new artifacts [S1].
*   **Artist Dependency:** Billboards require careful manual arrangement to look plausible from all angles, limiting their use in dynamic environments with changing lighting [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of billboards/particles. Rendering cost is low per instance, allowing millions of instances in theory, though practical limits depend on draw call overhead and texture memory [S1].
*   **Memory Complexity:** $O(N)$ for instance data plus texture memory. No volumetric grid storage is required [S1][S2, unofficial].
*   **Strengths:** Extremely low computational cost; suitable for real-time rendering of large-scale scenes (e.g., forests) or simple media effects [S1][S2, unofficial].
*   **Limitations:** No parallax (classical); popping artifacts; grazing angle artifacts (cross); artist-dependent; not suitable for dynamic lighting changes or complex geometry [S1][S2, unofficial].
*   **When to Use:** Real-time rendering of forests, simple participating media effects, or when performance is the primary constraint and visual fidelity can be compromised [S1][S2, unofficial].
*   **When Not to Use:** When parallax is required, for dynamic environments with changing lighting, or for high-fidelity volume rendering [S1][S2, unofficial].

## Background
Billboards have been a common tool for real-time rendering of forests since at least the late 1990s [S1]. They are widely used in industrial simulators and video games due to their efficiency [S1][S2, unofficial]. The concept of using particles rendered as billboards for participating media was explored in the context of real-time light transport [S2, unofficial]. Cross billboards were introduced to mitigate the lack of parallax in classical billboards, though they introduce their own artifacts [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S2] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
