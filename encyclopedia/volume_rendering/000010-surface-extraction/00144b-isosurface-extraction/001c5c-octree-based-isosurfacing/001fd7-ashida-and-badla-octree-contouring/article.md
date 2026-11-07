# Ashida and Badla Octree Contouring
Also called: Ashida and Badla's method

## Overview
Ashida and Badla Octree Contouring extracts contours directly from octrees with adaptive resolution by identifying cycles of contour faces intersecting each octree cell and creating one vertex for each cycle [S1, unofficial]. It sets itself apart from sibling methods by avoiding uniform grids or dual hexahedral meshes, though it is computationally expensive and may still produce non-manifold edges between neighboring cells [S1, unofficial].

## How it works
**Inputs:** Not given in the sources [S1, unofficial].
**Outputs:** Not given in the sources [S1, unofficial].
**Data Structures:** Not given in the sources [S1, unofficial].
**Steps:**
1. Traverse the octree to locate leaf cells intersecting the isosurface [S1, unofficial].
2. Identify cycles of contour faces intersecting each octree cell [S1, unofficial].
3. Create one mesh vertex for each identified cycle [S1, unofficial].
**Formulas:** Not given in the sources [S1, unofficial].
**Parameters:** Not given in the sources [S1, unofficial].
**Edge Cases and Pitfalls:** Non-manifold edges may still appear between neighboring cells despite claims of manifold output [S1, unofficial]. The cycle-identification process is time-consuming [S1, unofficial].

## Subtopics
[Manifold Vertex Clustering](2140-manifold-vertex-clustering.md): A topological constraint for vertex clustering that ensures the resulting mesh is a closed 2-manifold [S1, unofficial]. It sets itself apart from standard clustering by using topological checks (Euler characteristic and edge intersection counts) instead of relying solely on geometric error metrics like QEF [S1, unofficial]. Choose it when adaptive resolution is prioritized over strict manifold guarantees [S1, unofficial].

## Operating characteristics
* **Time Complexity:** High, due to the time-consuming cycle-identification process [S1, unofficial].
* **Memory Complexity:** Not given in the sources [S1, unofficial].
* **Measured Performance and Scaling:** Not given in the sources [S1, unofficial].
* **Strengths:** Adaptive resolution efficiently represents complex surfaces by operating directly on octrees [S1, unofficial].
* **Limitations:** May produce non-manifold edges between neighboring cells; high computational cost for cycle identification [S1, unofficial].
* **When to Use:** Scenes with highly adaptive features where uniform grids are inefficient [S1, unofficial].
* **When Not to Use:** When strict manifold guarantees are required without post-processing [S1, unofficial].

## Background
Proposed by Ashida and Badla as an alternative to contour simplification techniques used in Dual Contouring [S1, unofficial]. It is discussed in the context of manifold dual contouring and compared with other adaptive isosurface extraction techniques, such as Varadhan et al.'s feature-guided adaptive grid and Schaefer et al.'s dual hexahedral mesh approach [S1, unofficial].

## Sources
[S1] Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
