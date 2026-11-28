# View-Oriented Planes Technique
Also called: Planos orientados a la visión
## Overview
The View-Oriented Planes Technique is a volume rendering method that covers the entire data volume with a fixed number of oriented planes [S1, unofficial]. It differs from its parent [Plane Slicing](845-plane-slicing.md) by using planes oriented to the view rather than axis-aligned, and from [MC Slicing](15782-mc-slicing.md) by relying on a fixed plane count rather than isosurface extraction.
## How it works
**Inputs and outputs:**
- Input: Volume dimensions $W, H, D$ (in voxels).
- Output: A set of planes covering the volume.

**Data structures:**
- A list of $N$ planes. Each plane is defined by its position and orientation relative to the camera/view direction. The exact geometric definition of the plane parameters (normal, distance) is not specified in the sources [G: assumed standard plane equation $ax+by+cz+d=0$].

**Steps:**
1. Calculate the minimum number of planes required to intercept all possible voxels when the volume is rotated to its maximum plane size. The formula is:
   $$ N_{min} = \sqrt{W^2 + H^2 + D^2} $$
   [S1, unofficial]
2. Determine the actual number of planes $N$. The sources indicate that $N$ should be at least $N_{min}$ to ensure all voxels are intercepted. In the test configuration, $N=400$ was used for a volume of $181 \times 217 \times 181$ [S1, unofficial].
3. Generate the planes to cover the volume. The planes are oriented to the view. The exact algorithm for distributing these planes in 3D space is not detailed in the sources [G: likely evenly spaced along the view vector or distributed to cover the bounding box].
4. Render the planes using standard rasterization techniques, including texture mapping and blending.

**Parameters:**
- Number of planes $N$: Must be $\ge \sqrt{W^2 + H^2 + D^2}$. Typical value in tests: 400 for a $181 \times 217 \times 181$ volume [S1, unofficial].

**Edge cases and pitfalls:**
- Insufficient optimizations lead to low performance [S1, unofficial].
- High number of memory accesses makes alpha testing gains negligible [S1, unofficial].
- Increasing the number of planes improves image quality but significantly reduces speed due to increased fragments, memory accesses, and blending operations [S1, unofficial].
- Zoom level affects performance: higher zoom decreases frames per second [S1, unofficial].

## Subtopics

### [Multi-plane images (MPI)](21796-multi-plane-images-mpi.md)
Approaches that divide the plenoptic function into successive planes that store color and transparency information.

## Operating characteristics
**Time and memory complexity:**
- Not given in the sources.

**Measured performance:**
- The technique offers the lowest performance among compared methods due to insufficient optimizations [S1, unofficial].
- Using 400 planes for a $181 \times 217 \times 181$ volume significantly reduces speed [S1, unofficial].
- Performance decreases notably with higher zoom levels [S1, unofficial].

**Strengths and limitations:**
- Strength: Can achieve acceptable image quality with sufficient planes [S1, unofficial].
- Limitation: Low performance due to high memory access and blending overhead [S1, unofficial].

**When to use it and when not:**
- Use when view-oriented coverage is required and performance is not critical [S1, unofficial].
- Avoid when high performance is needed or when optimizations are not feasible [S1, unofficial].

## Background
The technique is described in a thesis on texture mapping based volume rendering [S1, unofficial]. It is presented as one of several volume rendering techniques, compared against others in terms of performance and image quality.

## Sources
[S1] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
