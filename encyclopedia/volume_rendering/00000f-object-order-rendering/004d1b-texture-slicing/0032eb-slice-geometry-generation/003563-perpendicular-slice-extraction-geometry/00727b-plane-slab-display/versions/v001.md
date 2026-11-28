# Plane Slab Display

Also called: Sequential Slab Display, Slab Stepping

## Overview
Plane Slab Display renders sequential slabs of volumetric data along a specified axis (x, y, or z) by stepping through grid indices [S1]. It solves the problem of visualizing a sequence of slices as a continuous animation or progression, allowing the user to traverse the volume depth [S1]. It differs from [AxisCut](18858-axiscut.md) by explicitly managing a slab thickness and a stepping increment per frame, rather than just defining a static cut plane [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume model, a specified axis ($x, y, \text{or } z$), and parameters: $start$, $end$, $increment$, and $depth$ [S1].
*   **Output:** A sequence of rendered frames, where each frame displays a slab of $depth$ planes [S1].

**Data Structures:**
The method relies on the grid index space of the volume. The `planes` command syntax is `axis[,start[,end[,increment[,depth]]]]` [S1].
*   $axis$: String, one of 'x', 'y', 'z' [S1].
*   $start$: Floating-point number, the index of the first displayed plane [S1].
*   $end$: Floating-point number, the stopping condition [S1].
*   $increment$: Floating-point number, the step size per frame [S1].
*   $depth$: Floating-point number, the number of planes in the slab [S1].

**Parameters and Defaults:**
*   Default $increment = 1$ [S1].
*   Default $depth = 1$ [S1].
*   If only $axis$ is provided, a single plane at the center of the current display region is shown [S1].
*   If $end$ is not supplied, only a single slab is shown [S1].

**Algorithm:**
1.  Initialize current index $i = start$ [S1].
2.  While $i \le end$:
    a.  Calculate the effective grid index $i_{eff} = \lfloor i / step \rfloor \times step$, where $step$ is the volume's grid step size [S1]. [G] Note: The source states indices are rounded down to a multiple of the step size.
    b.  Render a slab consisting of $depth$ planes starting at $i_{eff}$ along the specified $axis$ [S1].
    c.  Update $i = i + increment$ [S1].
3.  Stop when the next frame's index would exceed $end$ [S1].

**Edge Cases and Pitfalls:**
*   **Rounding Artifacts:** If $increment$ is smaller than the grid step size, $i_{eff}$ remains constant for multiple frames, causing the same plane to be displayed repeatedly [S1]. To avoid this, set the grid step size to 1 or ensure $start$, $end$, and $increment$ are consistent with the step size [S1]. For example, an increment of 0.25 with step size 1 shows each plane for 4 frames [S1].
*   **Single Plane Behavior:** If only the axis is given, the system defaults to a single plane at the center of the display region [S1].
*   **Missing End:** If $end$ is omitted, the loop terminates after the first slab [S1].

## Subtopics

### Expand Single Plane
Expands a single-plane display along its perpendicular axis to the full thickness of the data volume [S1]. This option does not apply to orthogonal planes [S1].

## Operating characteristics
Not given in the sources.

## Background
Origin: This method is part of the ChimeraX software suite, specifically within the `volume` command's `planes` options [S1]. It is used for interactive visualization of volumetric data, such as electron microscopy maps or medical imaging data, allowing users to step through the volume depth [S1].

## Sources
[S1] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
