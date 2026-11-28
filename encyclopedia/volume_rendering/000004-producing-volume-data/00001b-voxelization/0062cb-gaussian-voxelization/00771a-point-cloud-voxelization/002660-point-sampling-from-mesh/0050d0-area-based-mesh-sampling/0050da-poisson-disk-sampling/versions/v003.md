# Poisson Disk Sampling

Also called: Blue Noise Point Sampling

## Overview
Poisson disk sampling is a surface sampling method that uniformly distributes points on a mesh's surface with a minimum geodesic distance constraint of $2r$ between them, given a disk of radius $r$ [S1]. It utilizes blue noise properties to reduce the effects of aliasing in the later steps of the resulting model [S1]. It differs from its parent, Area-Based Mesh Sampling, by enforcing a strict minimum distance constraint rather than relying solely on area-proportional probability [S1].

## How it works
**Inputs:** A triangular mesh $M$; a sampling radius $r$ defining the exclusion zone [G].
**Outputs:** A set of points $P$ on the mesh surface [G].

**Data Structures:**
1. **Point Set $P$**: List of sampled points, each storing 3D coordinates $(x, y, z)$ and the triangle index $t$ where the point resides [G].
2. **Active List $A$**: Queue of points yet to be processed for neighborhood expansion [G].
3. **Spatial Hash Grid**: A grid with cell size $r$ to efficiently query neighbors [G].

**Steps:**
1. **Initialization**: Create an empty point set $P$ and active list $A$ [G].
2. **Seed Selection**: Select a random point $p_0$ on the mesh surface [G].
3. **Sampling Loop**: While $A$ is not empty:
   a. Remove a point $p$ from $A$ [G].
   b. Generate candidate points $c_i$ uniformly distributed within a region centered at $p$ such that they satisfy the minimum distance constraint [G].
   c. For each candidate $c_i$:
      i. Check if $c_i$ lies on the mesh surface. If not, discard [G].
      ii. Check if $c_i$ is at least $2r$ away (geodesic distance) from all points in $P$ [G]. Use the spatial hash grid to find candidate neighbors within Euclidean distance $2r$ [G]. For each neighbor, compute the geodesic distance [G]. If any neighbor is closer than $2r$, discard $c_i$ [G].
      iii. If valid, add $c_i$ to $P$, add $c_i$ to $A$, and update the spatial hash grid [G].
4. **Termination**: When $A$ is empty, return $P$ [G].

**Formulas:**
- Minimum distance constraint: $d_{min} = 2r$ [S1].
- Exclusion zone radius: $r$ [S1].

**Parameters:**
- $r$: Sampling radius. Typical values depend on mesh scale. Not given in the sources.
- $k$: Number of candidates per active point. Typical value: 30 [G].

**Edge Cases:**
- **Precision**: Geodesic distance computation can be sensitive to mesh quality. Degenerate triangles may cause failures [G].
- **Boundaries**: On open meshes, points near boundaries may have fewer valid sampling locations [G].
- **Empty Mesh**: If the mesh has no area, return empty set [G].

## Subtopics

### [Fast Poisson Disk Sampling (2D)](16881-fast-poisson-disk-sampling-2d.md)
Generates points within a 2D domain with a minimum distance constraint $r$, using a spatial grid and active list for linear time complexity. Chosen when 2D blue noise is needed rather than 3D surface sampling.

## Operating characteristics
- **Time Complexity**: Not given in the sources [G].
- **Memory Complexity**: Not given in the sources [G].
- **Strengths**: Produces blue noise distribution, reducing aliasing artifacts in rendering [S1].
- **Limitations**: Geodesic distance computation is slow. May fail to fill small regions if $r$ is too large relative to local geometry [G].
- **When to use**: When blue noise properties are critical for reducing aliasing in volume rendering or texture mapping [S1].
- **When not to use**: When performance is critical and blue noise is not required; use Area-Based Mesh Sampling instead [S1].

## Background
Poisson disk sampling was introduced as a method for generating blue noise point distributions [S1]. It is widely used in computer graphics for texture synthesis, point cloud generation, and anti-aliasing [S1]. The method ensures that points are well-spaced while maintaining a random appearance, which is desirable for avoiding regular grid artifacts [S1].

## Sources
[S1] A Comparative Assessment Of Implicit and Explicit Plenoptic Scene Representations, https://inria.hal.science/hal-04709909v1, fetched 2026-10-26
