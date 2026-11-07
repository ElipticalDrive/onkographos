# BasicDAGUncompressedColors


## Overview
BasicDAGUncompressedColors is a leaf data storage strategy for Sparse Voxel DAGs that stores color values in a flat, uncompressed array indexed by leaf count [S1, unofficial]. It differs from sibling variants by decoupling color storage from the geometric DAG structure, allowing for simple direct indexing but requiring separate traversal logic for color retrieval in some implementations [S1, unofficial].

## Shared principles
The family stores color data for DAG leaves in a flat array. The index into this array corresponds to the sequential count of leaves encountered during a traversal or build process. The data type for colors is typically a 32-bit unsigned integer (`uint32`), representing a packed color value (e.g., RGBA) [S1, unofficial].

## Subtopics

### BasicDAGUncompressedColors
Stores uncompressed color data for DAG leaves in a flat array, indexed by leaf count [S1, unofficial]. The layout consists of a `ColorLeaf` struct containing a `StaticArray<uint32> colors`, where each element stores a color value [S1, unofficial]. Retrieval is performed via a function `get_color(leaveCount)` which returns an `UncompressedColor` struct wrapping the color value at index `leaveCount` in the colors array: `{ colors[leaveCount] }` [S1, unofficial]. This method is chosen when simplicity of access is prioritized over memory compactness or when the color data is already uncompressed in the source format.

### HashDAG Color Lookup
A GPU kernel implementation that retrieves the color for a pixel by traversing the DAG path and looking up the color in a separate color DAG [S1, unofficial]. This variant is distinct because it performs traversal during the rendering pass rather than relying on a pre-indexed flat array for direct access.

**Algorithm:**
1.  **Input:** `traceParams` containing `pathsSurface`, `colorsSurface`, `toolInfo`, `debugColors`, and `debugColorsIndexLevel` [S1, unofficial].
2.  **Initialization:** Calculate pixel coordinates. If outside bounds, return [S1, unofficial]. Load path from `pathsSurface`; if null, set default color `(187, 242, 250)` and return [S1, unofficial]. Initialize traversal: `level=0`, `nodeIndex=root`, `colorNodeIndex=0`, `colorLeaf=default` [S1, unofficial].
3.  **Traversal Loop:** While `level < leaf_level`:
    *   Increment `level` [S1, unofficial].
    *   Get node and `childMask`, determine child index from path [S1, unofficial].
    *   If child not in `childMask`, set magenta color `(0xFF00FF)` and return [S1, unofficial].
    *   If `level-1 < color_tree_levels`, update `colorNodeIndex` and get `colorLeaf` if at final color tree level [S1, unofficial].
    *   Handle debug colors: Index, Position, ColorTree, ColorBits, MinColor, MaxColor, Weight [S1, unofficial].
    *   If at leaf level, count preceding leaves to determine `nof_leaves` [S1, unofficial].
    *   Update `nodeIndex` to child [S1, unofficial].
4.  **Finalization:**
    *   If `colorLeaf` invalid or index out of range, set invalid color and return [S1, unofficial].
    *   Get compressed color from `colorLeaf` using `nof_leaves` [S1, unofficial].
    *   Extract final color based on debug mode or default color [S1, unofficial].
    *   Write color to `colorsSurface` [S1, unofficial].

This variant is chosen when the rendering architecture requires on-the-fly color resolution from a separate color tree structure, potentially allowing for different compression schemes or hierarchical color data independent of the geometric leaf count.

## Background
The BasicDAGUncompressedColors structure and its variants, particularly the HashDAG Color Lookup, are documented in the context of the HashDAG project by Phyronnaz [S1, unofficial]. The implementation details, including benchmark scripts for memory overhead and page size tuning, are found in the associated GitHub repository [S1, unofficial]. The specific GPU kernel logic for HashDAG Color Lookup suggests its use in real-time volume rendering pipelines where path-based traversal is already performed for geometry [S1, unofficial].

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
