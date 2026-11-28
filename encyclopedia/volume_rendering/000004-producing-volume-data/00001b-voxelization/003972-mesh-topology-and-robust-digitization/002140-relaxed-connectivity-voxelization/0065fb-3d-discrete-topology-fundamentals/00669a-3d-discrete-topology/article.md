# 3D Discrete Topology


## Overview
3D Discrete Topology is the theoretical framework for defining topological properties, such as surface separation and connectivity, within a 3D discrete voxel grid [S1, unofficial]. It solves the problem of "discrete disagreement," where standard continuous topology concepts fail in discrete space (e.g., 8-connected black curves can be penetrated by 8-connected white pixels without intersection) [S1, unofficial]. It sets itself apart from general graph theory by specifically addressing the geometric adjacency constraints of cubic voxels in 3D Euclidean space to ensure robust surface representation for volume rendering and ray tracing [S1, unofficial].

## Shared principles
The foundation of 3D Discrete Topology rests on the definition of the discrete space and the adjacency relations between voxels.

**Space Definition**
The 3D discrete space is defined as a set of integral grid points in 3D Euclidean space, specified by Cartesian coordinates $(x, y, z)$ [S1, unofficial]. A voxel is the unit cubic volume centered at each integral grid point [S1, unofficial].

**Voxel Values**
In the binary case, voxel values are mapped to the set $\{0, 1\}$ [S1, unofficial]:
*   **Black voxels (1):** Represent opaque objects [S1, unofficial].
*   **White voxels (0):** Represent the transparent background [S1, unofficial].

**Adjacency and Connectivity**
Adjacency is defined by the geometric sharing of boundaries between voxels. The prefix $N$ denotes the type of adjacency, where $N \in \{6, 18, 26\}$ [S1, unofficial]:

1.  **6-adjacency:** Two voxels are 6-adjacent if they share a face [S1, unofficial].
2.  **18-adjacency:** Two voxels are 18-adjacent if they share an edge or a face [S1, unofficial].
3.  **26-adjacency:** Two voxels are 26-adjacent if they share a vertex, an edge, or a face [S1, unofficial].

Every voxel has exactly 26 adjacent neighbors:
*   8 voxels share a vertex (corner) [S1, unofficial].
*   12 voxels share an edge [S1, unofficial].
*   6 voxels share a face [S1, unofficial].

**Connectivity and Paths**
*   **N-path:** A sequence of voxels having the same value (e.g., all black) where every consecutive pair of voxels is $N$-adjacent [S1, unofficial].
*   **N-connected:** A set of voxels $W$ is $N$-connected if there exists an $N$-path between every pair of voxels in $W$ [S1, unofficial].
*   **N-connected component:** A maximal $N$-connected set [S1, unofficial].

**Opposite Connectivity**
To prevent topological inconsistencies (such as penetration), opposite connectivity types are used for foreground and background [S1, unofficial]:
*   In 2D, 4 and 8 are opposite.
*   In 3D, 6 is opposite to 26 or to 18 [S1, unofficial].

**Separation and Minimality**
*   **N-separating:** A subset $S$ of black voxels is $N$-separating if it is not $N$-connected, meaning it consists of at least two white $N$-connected components [S1, unofficial].
*   **N-simple voxel:** Let $W$ be an $N$-separating surface. A voxel $p \in W$ is $N$-simple if removing it ($W \setminus \{p\}$) results in a set that is still $N$-separating [S1, unofficial].
*   **N-minimal surface:** An $N$-separating surface is $N$-minimal if it contains no $N$-simple voxels [S1, unofficial].

**Covering**
*   **Cover:** A set of voxels is a cover of a continuous surface if every point of the continuous surface lies within a voxel of the cover [S1, unofficial].
*   **Minimal cover:** A cover is minimal if none of its subsets is also a cover [S1, unofficial].

**Discrete Disagreement**
A key phenomenon in 3D discrete topology is "discrete disagreement" [S1, unofficial]. In 2D, an 8-connected black curve cannot be penetrated by 8-connected white pixels without intersection. However, in 3D, this property does not hold analogously; specific connectivity configurations allow penetration without intersection, which is why opposite connectivity types are required [S1, unofficial].

**Limitations**
*   A cover is not necessarily separating [S1, unofficial].
*   Even a minimal cover is not necessarily $N$-minimal for any $N$ [S1, unofficial].

## Subtopics

### 3D Discrete Topology Fundamentals
*   **What it is:** The foundational definitions of 3D discrete space, voxel geometry, and basic adjacency relations.
*   **Distinction:** Provides the static geometric and set-theoretic definitions (coordinates, unit cubes, basic adjacency counts) without addressing dynamic topological properties like separation or minimality.
*   **When to choose:** Use this as the starting point for understanding the coordinate system and voxel structure.
*   **Link:** [3D Discrete Topology Fundamentals](19505-3d-discrete-topology-fundamentals.md)

## Background
The theory of 3D discrete topology was formalized to address topological issues in 3D discrete space, such as defining the separation of both sides of a surface, which cannot be well-defined by employing 2D terminology [S1, unofficial]. It originated in the context of volume visualization and voxelization, specifically to support applications like fast ray tracing where space subdivision is employed [S1, unofficial]. The concepts were detailed by Arie Kaufman in the chapter "Fundamentals of 3D Discrete Topology" within the book *Volume Visualization: Principles and Advances* [S1, unofficial]. The theory is essential for binary voxelization techniques that aim to generate $N$-separating, $N$-minimal, and covering surfaces [S1, unofficial].

## Sources
[S1] Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
