# Mesh Extraction

Also called: Disjoint Patch MC, MC with edge grouping

## Overview
Mesh Extraction converts the Patch-Grid implicit representation into a valid triangle mesh by restricting surface generation to specific grid cells [S1]. It sets itself apart from standard Marching Cubes by applying the algorithm only to cells containing a single surface patch or to a subdivided merge grid for intersecting regions, thereby ensuring validity through patch and merge loss constraints [S1].

## How it works
**Inputs:** Patch volumes enclosing a shape composed of multiple surface patches [S1].
**Outputs:** A valid triangle mesh representation [S1].
**Data Structures:**
*   **Patch Volume Cell:** A grid cell associated with one or more surface patches [S1].
*   **Merge Grid:** An octree structure used to subdivide grid cells where two or more patches intersect [S1].
*   **Exact Layouts (fields, sizes, bit packing):** Not given in the sources [S1].
**Steps:**
1. Enclose the given shape composed of multiple surface patches with patch volumes [S1].
2. Identify overlapped grid cells in relevant patch volumes where two or more patches intersect, and subdivide them into a merge grid [S1].
3. Perform the Marching Cubes algorithm to mesh the zero-level sets in the patch volume cells which contain only a single surface patch [S1].
4. For patch volume cells containing intersecting patches, only mesh the zero-level sets of the patches in the merge cells [S1].
5. Extract surfaces only from cells where patch loss and merge loss are enabled to ensure mesh validity [S1].
**Formulas:** Not given in the sources [S1].
**Parameters:**
*   **Maximal Merge Grid Depth:** The maximum recursion depth for the merge grid octree, empirically set to 8 [S1].
**Edge Cases and Pitfalls:**
*   **Thin Geometric Features:** Narrow slits and thin solids are challenging for global approaches; this method isolates patches to handle them [S1].
*   **Intersection Handling:** Complex intersections require subdivision into the merge grid to maintain topological correctness [S1].

## Subtopics

### Mesh Extraction via Patch Volumes and Merge Grid
This is the specific implementation of Mesh Extraction within the Patch-Grid framework [S1]. It explicitly handles intersecting surface patches by subdividing overlapped grid cells into a merge grid [S1]. It sets itself apart by restricting Marching Cubes to single-patch cells and merge-grid zero-level sets, ensuring validity via patch and merge loss constraints [S1]. Choose it when extracting meshes from multi-patch implicit representations with intersections or thin geometric features [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Ensures mesh validity in the presence of intersecting patches and effectively handles thin geometric features like narrow slits and thin solids [S1].
*   **Limitations:** Requires the computation and storage of a merge grid octree for intersecting regions [S1].
*   **When to use:** When extracting valid triangle meshes from shapes composed of multiple intersecting surface patches or containing thin geometric features [S1].
*   **When not:** Not given in the sources [S1].

## Background
The method was introduced in "Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation" [S1]. It addresses the challenges of modeling thin geometric features (narrow slits, thin solids) and open surface boundaries, which are difficult for global neural implicit representations like NH-Rep or NGLOD [S1]. The approach builds on decomposing complex shapes into simpler patch volumes and using a merge grid to handle intersections [S1].

## Sources
[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
