# Marker Placement
Also called: Cone Markers

## Overview
Marker Placement is an illustrative volume rendering technique that overlays cone-shaped geometric primitives into the 3D data space to aid feature location and facilitate discussion during parameter adjustments [S1, S2, unofficial]. It distinguishes itself from sibling methods like [Boxes](9605-visual-conventions.md) by using directional, non-occluding shapes that can be arbitrarily oriented to indicate specific vectors or points of interest, rather than providing bounding context [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a user interaction stream (pointer/wand position and orientation), and a set of existing markers.
*   **Output:** A rendered image containing the volume data and the cone markers.

**Data Structures:**
Each marker is defined by:
1.  **Position:** A 3D vector $(P_x, P_y, P_z)$ representing the base or center of the cone in world/voxel coordinates.
2.  **Orientation:** A 3D vector $(O_x, O_y, O_z)$ or a quaternion representing the cone's axis of symmetry.
3.  **Scale:** A scalar value $S$ controlling the height and base radius of the cone.
4.  **Color:** A 4-tuple $(R, G, B, A)$ for rendering.

**Steps:**
1.  **Placement:** The user selects a location in the 3D space using a pointer or wand [S1, S2, unofficial].
2.  **Instantiation:** A cone marker is created at the selected position [S2, unofficial].
3.  **Manipulation:** The user moves and rotates the marker using the pointer/wand [S1, S2, unofficial]. The system updates the marker's position and orientation vectors in real-time [S1, S2, unofficial].
4.  **Rendering:** The cone is rendered as an opaque or semi-transparent object within the volume rendering pipeline. The cone is typically rendered after the volume slice or ray-marched volume to ensure it appears as an overlay [S1, S2, unofficial].

**Formulas:**
The geometry of a cone marker with height $H$ and base radius $R$ is defined by its axis vector $\vec{A}$ and base center $\vec{C}$. For a point $\vec{p}$ on the cone surface:
$$ \vec{p}(t, \theta) = \vec{C} + t \cdot \vec{A} + R \cdot (1 - \frac{t}{H}) \cdot (\cos(\theta)\vec{u} + \sin(\theta)\vec{v}) $$
where $t \in [0, H]$, $\theta \in [0, 2\pi)$, and $\vec{u}, \vec{v}$ are orthogonal vectors perpendicular to $\vec{A}$ [G].

**Parameters:**
*   **Cone Height/Radius Ratio:** Not specified in sources [G].
*   **Color:** Typically high-contrast (e.g., red, yellow) to stand out against volume data [G].

**Edge Cases:**
*   **Occlusion:** Markers may be occluded by opaque volume regions. Transparency settings for the volume or markers may be required [G].
*   **Precision:** Marker positions must be mapped accurately from screen/pointer space to 3D voxel space to avoid misalignment [G].

## Subtopics

### Cone Markers
The specific geometric shape used. It provides directional cues unlike [Boxes](9605-visual-conventions.md). Choose when orientation matters. [S1, S2, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(1)$ per marker for update; $O(M)$ for rendering $M$ markers.
*   **Memory Complexity:** $O(M)$ to store marker states.
*   **Strengths:** Intuitive for pointing out specific features; supports arbitrary orientation; lightweight [S1, S2, unofficial].
*   **Limitations:** Can clutter the view if overused; does not provide spatial extent context like [Boxes](9605-visual-conventions.md) [S1, S2, unofficial].
*   **When to Use:** Interactive exploration, presentations, highlighting specific points or vectors [S1, S2, unofficial].
*   **When Not to Use:** When spatial extent or bounding information is needed [S1, S2, unofficial].

## Background
Originated in the context of the IEEE Visualization 2004 Contest for user-friendly volume data set exploration, specifically in systems utilizing immersive environments like the Cave [S1, S2, unofficial]. It was designed to aid discussion and feature re-location after parameter changes [S1, S2, unofficial].

## Sources
[S1] IEEE Visualization 2004 Contest: User Friendly Volume Data Set Exploration in the Cave, http://vis.computer.org/vis2004contest/brown/, fetched 2026-10-28 (unofficial)
[S2] IEEE Visualization 2004 Contest: User Friendly Volume Data Set Exploration in the Cave, http://web.eng.ucsd.edu/~jschulze/projects/vis04-contest/, fetched 2026-10-28 (unofficial)
