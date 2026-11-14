# Dig Tool
Also called: 3D MoleView, Radial Displacement Tool

## Overview
The Dig Tool alleviates occlusion in volumetric datasets by smoothly displacing data points radially away from a user-defined focus point, creating a gap to reveal hidden internal structures [S1]. It serves as the 3D equivalent of the MoleView principle [S1], operating on screen-space projections to manipulate the visible density field without altering the underlying geometry permanently until release [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of visible data points $P$, a focus point $x \in \mathbb{R}^2$ in screen space, a radius $r \in \mathbb{R}^+$, and the view plane normal $n$.
*   **Output:** A temporary displaced configuration of points $p(t)$ that reveals occluded structures, reverting to the original configuration upon interaction release [S1].

**Data Structures:**
*   **Point Set:** A list or array of 3D coordinates representing the visible data points (e.g., isosurface vertices or ray-cast sample positions) [S1].
*   **Focus Disk $C$:** Implicitly defined as the set of points $p$ whose screen projection lies within a circle of radius $r$ centered at $x$ [S1].

**Parameters:**
*   **Radius $r$:** Controlled by the mouse wheel [S1].
*   **Time Steps:** The interpolation is performed over 50 to 100 discrete time steps [S1].

**Algorithm Steps:**
1.  **Selection:** Identify all points $p \in P$ such that the screen-space projection of $p$ falls within the disk $C$ of radius $r$ centered at the focus point $x$ [S1].
2.  **Displacement Vector Calculation:** For each selected point $p$, compute the shortest vector $v$ from the view ray passing through $x$ to $p$ [S1]:
    $$ v = p - x - ((p - x) \cdot n)n $$
    where $n$ is the view plane normal [S1].
3.  **Target Position Calculation:** Compute the target displacement position $p_{disp}$ on the boundary of the disk $C$ in the direction of $v$:
    $$ p_{disp} = r \frac{v}{\|v\|} $$
    [S1]. Note: If $\|v\| \approx 0$, the point is at the focus; the direction is undefined, but such points are typically handled by the interpolation logic directly or ignored if they do not contribute to the gap [G].
4.  **Forward Interpolation (Digging):** While the mouse button is pressed, update the position of each selected point $p$ towards $p_{disp}$ using linear interpolation over time $t \in [0, 1]$:
    $$ p(t) = (1 - t)p + t p_{disp} $$
    The parameter $t$ is incremented in small steps (50–100 steps total) to create a smooth animation [S1].
5.  **Inverse Interpolation (Reset):** Upon mouse button release, perform inverse interpolation from the current displaced position $p(t)$ back to the original location $p$. This causes the points to "snap back" to their original locations [S1].

**Pitfalls and Edge Cases:**
*   **Velocity Gradient:** Points close to the focus $x$ drift quickly to the boundary $\partial C$, while points near $\partial C$ move slower. This creates a clear gap at the center and progressively weaker deformation towards the edge [S1].
*   **Occlusion Locking:** The tool only affects unlocked density values. High or low density values may be locked, meaning only average density values (e.g., skin) or specific ranges are displaced [S1].

## Operating characteristics
*   **Strengths:** Effectively reveals hidden structures by creating a temporary void around the focus point [S1]. Useful for forecasting the effect of subsequent brushing actions, as pushed items are the unlocked ones affected by further interactions [S1].
*   **Limitations:** The deformation is purely visual/interactive and does not modify the underlying dataset permanently unless explicitly committed by the application [S1]. Performance depends on the number of visible points within the radius $r$ [S1].
*   **When to use:** When exploring dense volumetric datasets where occlusion hides critical internal features [S1].
*   **When not to use:** When precise quantitative analysis of the displaced region is required, as the geometry is temporarily altered [G].

## Background
The Dig Tool was proposed in the context of "Color Tunneling" for interactive exploration and selection in volumetric datasets [S1]. It is conceptually derived from the MoleView principle [18], which operates in 2D, extending the concept to 3D volumetric data [S1]. The tool is part of a suite including Brush and Lock tools for managing visibility and selection in medical imaging and other domains [S1].

## Sources
[S1] Color Tunneling: Interactive Exploration and Selection in Volumetric Datasets, https://doi.org/10.1109/pacificvis.2014.61, fetched 2026-10-26
