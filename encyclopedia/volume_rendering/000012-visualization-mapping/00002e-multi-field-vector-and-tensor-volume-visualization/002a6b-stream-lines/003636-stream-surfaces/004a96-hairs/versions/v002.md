# Hairs


## Overview
Hairs is a visualization technique for representing steady-state flow fields near or around contour surfaces, specifically addressing the failure of fiber bundles (or line bundles) when the flow vector points slightly into the surface [S1][S2, unofficial]. It solves this by growing tiny, segmented hairs from the surface that bend according to the local velocity field [S1][S2, unofficial]. Unlike its sibling [Stream Surfaces](7696-stream-surfaces.md), which stitches adjacent streamlines into a continuous surface, Hairs remains a discrete set of line segments, offering a lighter-weight alternative for sparse or near-surface flow visualization where surface continuity is difficult to maintain or unnecessary [S1][S2, unofficial].

## How it works
**Inputs:**
*   A 3D velocity vector field $\mathbf{V}$.
*   A scalar field $\phi$ defining the contour surface $\phi(\mathbf{x}) = 0$.
*   Surface normal field $\mathbf{N}$ (derived from $\nabla \phi$).
*   User parameters: number of segments $N_{seg}$, velocity scale $S_v$, segment weights $w_i$, interpolation coefficients $t_i$, root color $\mathbf{C}_{root}$, head color $\mathbf{C}_{head}$, jitter scales $\sigma_{H,S,V}$, and transparency values $\alpha_i$.

**Outputs:**
*   A set of rendered hairs, each defined by a sequence of vertices and associated colors/transparency.

**Data Structures:**
Each hair is a list of $N_{seg} + 1$ vertices $P_0, P_1, \dots, P_{N_{seg}}$.
Each vertex $P_i$ has an associated color $\mathbf{C}_i$ (in RGB or HSV) and transparency $\alpha_i$.
The splat positions are derived from the contour surface.

**Steps:**

1.  **Splat Positioning:**
    Scatter splats near the contour surface. The method uses Levoy's method to position splats near the contour surface [S1][S2, unofficial]. To ensure they originate from the surface, move each splat position $\mathbf{p}$ towards the contour surface by a multiple of the gradient $\nabla \phi$ [S2, unofficial]. Let the final base position be $\mathbf{P}_0$.

2.  **Hair Generation (Per Hair):**
    Initialize the current position $\mathbf{P}_{curr} = \mathbf{P}_0$.
    For each segment index $i$ from $0$ to $N_{seg} - 1$:
    
    a. **Direction Calculation:**
    Retrieve the normalized surface normal $\mathbf{n} = \frac{\nabla \phi(\mathbf{P}_{curr})}{\|\nabla \phi(\mathbf{P}_{curr})\|}$ and the velocity vector $\mathbf{v} = \mathbf{V}(\mathbf{P}_{curr})$ [S1][S2, unofficial].
    Calculate the direction vector $\mathbf{d}_i$ using the formula:
    $$ \mathbf{d}_i = w_i \left( t_i \mathbf{n} + (1 - t_i) (\mathbf{v} \cdot S_v) \right) $$
    Where:
    *   $w_i$ is the weighting factor for segment $i$ (controls apparent stiffness) [S1][S2, unofficial].
    *   $t_i$ is the user-specified interpolation coefficient at segment point $i$ [S1][S2, unofficial].
    *   $S_v$ is the user-specified velocity scale parameter [S1][S2, unofficial].
    *   $\mathbf{n}$ is normalized [S1][S2, unofficial].
    *   $\mathbf{v}$ is scaled by $S_v$ [S1][S2, unofficial].
    
    b. **Vertex Update:**
    Compute the next vertex:
    $$ \mathbf{P}_{i+1} = \mathbf{P}_{curr} + \mathbf{d}_i $$
    Set $\mathbf{P}_{curr} = \mathbf{P}_{i+1}$ [S1][S2, unofficial].

    c. **Color Calculation:**
    Determine the color $\mathbf{C}_i$ for vertex $i$.
    1.  Define $\mathbf{C}_{splat}$ by mapping the scalar field value at $\mathbf{P}_i$ to a color table [S1][S2, unofficial].
    2.  The user specifies fractions $f_{root}, f_{splat}, f_{head}$ for the endpoint of segment $i$ [S1][S2, unofficial].
    3.  Compute the base color in RGB:
        $$ \mathbf{C}_{base} = f_{root} \mathbf{C}_{root} + f_{splat} \mathbf{C}_{splat} + f_{head} \mathbf{C}_{head} $$
    4.  Convert $\mathbf{C}_{base}$ to HSV space [S1][S2, unofficial].
    5.  Add random jitter to the HSV components:
        *   Hue $H$: Add random noise. If $H < 0$, wrap to $H + 1$; if $H > 1$, wrap to $H - 1$ [S1][S2, unofficial].
        *   Saturation $S$: Clamp to $[0, 1]$ [S1][S2, unofficial].
        *   Value $V$: Clamp to $[0, 1]$ [S1][S2, unofficial].
        The jitter magnitude is controlled by scale factors $\sigma_H, \sigma_S, \sigma_V$ [S1][S2, unofficial].
    6.  Convert the jittered HSV back to RGB to get $\mathbf{C}_i$ [S1][S2, unofficial].

    d. **Transparency Assignment:**
    Assign the user-specified transparency $\alpha_i$ to vertex $i$. No jitter is applied to transparency [S1][S2, unofficial].

3.  **Rendering:**
    Render the line segments connecting $\mathbf{P}_i$ to $\mathbf{P}_{i+1}$ using the colors $\mathbf{C}_i$ and $\mathbf{C}_{i+1}$ and transparency $\alpha_i$ [S1][S2, unofficial].

**Parameters:**
*   $N_{seg}$: Number of connected line segments per hair. Default is 6 [S1][S2, unofficial].
*   $S_v$: Velocity scale factor [S1][S2, unofficial].
*   $w_i$: Weighting factor per segment. Greater weights on initial segments favor the normal direction; lower weights on middle segments; large weights on the last one or two segments favor the velocity direction for longer hairs [S1][S2, unofficial].
*   $t_i$: Interpolation coefficients at each line segment point, specified by the user. These determine the blend between normal and velocity directions [S1][S2, unofficial].
*   $\mathbf{C}_{root}$: User-specified root color [S1][S2, unofficial].
*   $\mathbf{C}_{head}$: User-specified vector head color [S1][S2, unofficial].
*   $\sigma_{H,S,V}$: Scale factors for HSV jitter [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Flow into Surface:** If the flow is strongly into the surface, the normal vector might be ill-defined or the hair might intersect the surface immediately. The method assumes the flow is "slightly into" or near the surface [S1][S2, unofficial].
*   **Precision:** HSV hue wrapping must be handled correctly to avoid discontinuities in color [S1][S2, unofficial].
*   **Degenerate Input:** If $\nabla \phi = 0$, the normal is undefined. [G] In practice, a fallback normal (e.g., view vector or previous normal) should be used, though not specified in the source.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of hairs and segments. Each hair requires $N_{seg}$ vector field and gradient evaluations.
*   **Memory Complexity:** Linear with respect to the number of hairs. Each hair stores $N_{seg}+1$ vertices and colors.
*   **Strengths:** Effective for visualizing flow near surfaces where fiber bundles fail due to inward flow. Provides clear indication of flow direction and magnitude via bending [S1][S2, unofficial].
*   **Limitations:** Discrete nature may not convey smooth flow topology as well as stream surfaces. Performance degrades with very high hair density [S1][S2, unofficial].
*   **When to use:** Near-surface flow visualization, especially when flow vectors point into the surface [S1][S2, unofficial].
*   **When not to use:** When a continuous surface representation is required or when flow is far from any surface [S1][S2, unofficial].

## Background
Hairs were introduced by Brian Cabral and Lieth Leedom in their work on visualizing 3D velocity fields near contour surfaces [S1][S2, unofficial]. The technique was developed to address limitations of fiber bundles in specific flow scenarios near surfaces [S1][S2, unofficial]. It is part of the broader family of vector field visualization techniques, building on [Stream Surfaces](7696-stream-surfaces.md) and [Stream-Balls](7700-stream-balls.md) [S1][S2, unofficial].

## Sources
[S1] Visualizing 3D velocity fields near contour surfaces, https://doi.org/10.5555/951087.951133, fetched 2026-10-26
[S2] Visualizing 3D velocity fields near contour surfaces. Revision 1, https://digital.library.unt.edu/ark:/67531/metadc1395286/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
