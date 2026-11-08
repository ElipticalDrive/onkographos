# Gray Transform (G_o, G_s)

Also called: Surface to Density Conversion

## Overview
The Gray Transform converts an object or surface description into a scene (density volume) suitable for visualization or manufacturing [S1, unofficial]. It distinguishes itself from sibling methods like [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) by explicitly defining density values based on interior/exterior status and distance, rather than directly computing alpha during ray marching. It serves as the foundational step for subsequent operations like filtering and interpolation [S1, unofficial].

## How it works
**Inputs:**
- An object $Q$ (set of voxels) or a closed surface $S$.
- A scene domain $V$ containing the object/surface.
- Parameters: $H$ (high density for interior), $L_0$ (base density for exterior), $L$ (low density, often $L_0$).

**Outputs:**
- A scene $(V, f)$ where $f(v)$ is the density at voxel $v$.

**Data Structures:**
- $V$: A 3D grid (volume) of voxels.
- $f$: A scalar field mapping each $v \in V$ to a density value.

**Steps:**
1. **For Object $Q$ ($G_o$):**
   - Iterate over all voxels $v$ in domain $V$.
   - If $v \in Q$, set $f(v) = H$.
   - If $v \notin Q$, set $f(v) = L$ (or $L_0$).
   - Formula [S1, unofficial]:
     $$
     f(v) = \begin{cases} 
     H & \text{if } v \in Q \\
     L & \text{if } v \in V - Q
     \end{cases}
     $$

2. **For Surface $S$ ($G_s$):**
   - Verify $S$ is closed (partitions space into interior and exterior) [S1, unofficial].
   - For each voxel $v \in V$:
     - Compute distance $d_l(v, S)$ from $v$ to surface $S$. This can be 2D distance in the slice containing $v$ or 3D distance [S1, unofficial].
     - Determine if $v$ is in the interior, exterior, or intersecting $S$.
     - Assign density:
       $$
       f(v) = \begin{cases} 
       d_l(v, S) & \text{if } v \text{ is in the interior of } S \\
       L_0 - d_l(v, S) & \text{if } v \text{ is in } V \text{ but in the exterior of } S \\
       L_0 & \text{if } v \text{ intersects } S
       \end{cases}
       $$
   - Note: The source uses $d_l(v, S)$ for interior and $L_0 - d_l(v, S)$ for exterior [S1, unofficial]. This implies density decreases as one moves away from the surface into the exterior, and increases as one moves into the interior? Or vice versa? The formula is explicit: interior = distance, exterior = $L_0$ - distance. If $d_l$ is small near surface, interior density is small, exterior is near $L_0$. This seems counter-intuitive for "solid" objects where interior is usually high density. However, we must follow the source [S1, unofficial]. Let's re-read carefully: "$f(v) = H$ if $v \in V$ is in the interior... and $f(v) = L_0$ if $v \in V$ is not in the interior." This is for $G_o$. For $G_s$, it says "$d_l(v, S)$ if interior... $L_0 - d_l(v, S)$ if exterior". This creates a gradient.

**Parameters:**
- $H$: High density value. Typical value: Not given in the sources.
- $L_0$: Base density value. Typical value: Not given in the sources.
- $L$: Low density value. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- The surface $S$ must be closed for $G_s$ to be defined. If $S$ does not partition space into interior and exterior, $G_s$ is undefined [S1, unofficial].
- Precision in determining "interior" vs "exterior" vs "intersecting" is critical. A precise definition of these notions in digital space is required [S1, unofficial].

## Subtopics

### Object/Surface Filtering Composite
This composite operator combines Gray Transform with filtering. It converts an object $Q$ to a scene $G_o(Q)$ using equation 29 (hard assignment), filters the scene using a filter $F_s$, and then determines the filtered object $Q'$ from the filtered scene [S1, unofficial]. If $F_s$ is a smoothing filter, a threshold operator $O_s$ is applied, with the threshold determined from knowledge of $L$ and $H$ in equation 29 [S1, unofficial]. Similarly, a surface $S$ can be converted to $G_s(S)$, filtered, and then a surface detector $B_s$ extracts the filtered surface $S'$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the size of the domain $V$ and the cost of computing distance $d_l(v, S)$. For $G_o$, it is $O(|V|)$. For $G_s$, it is $O(|V| \cdot \text{cost of distance})$.
- **Memory Complexity:** $O(|V|)$ to store the density field $f$.
- **Strengths:** Provides a clear physical model of surfaces/objects for manufacturing (e.g., prosthesis design) [S1, unofficial]. Enables subsequent operations like interpolation and filtering [S1, unofficial].
- **Limitations:** Requires a closed surface for $G_s$. The resulting density field may not be directly suitable for volume rendering without further processing (like alpha computation) [S1, unofficial].
- **When to use:** When converting geometric descriptions to volumetric data for manufacturing or as a preprocessing step for filtering/interpolation [S1, unofficial].

## Background
The Gray Transform was introduced in the context of imaging transforms for visualizing surfaces and volumes [S1, unofficial]. It is part of a battery of structure-related operations available for digital entities, particularly in applications like prosthesis design and radiation therapy planning [S1, unofficial]. The concept relies on the digitization of parametrically defined continuous geometric objects into digital object representations [S1, unofficial].

## Sources
[S1] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29 (unofficial)
