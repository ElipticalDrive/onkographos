# 4D Block Geometry Slicing

Also called: 4D cross-section rendering, Hyperplane block intersection

## Overview
4D Block Geometry Slicing renders a three-dimensional cross-section (slice) of a four-dimensional voxel world by intersecting 4D primitives with a 3D hyperplane. It solves the problem of visualizing higher-dimensional spatial data by generating accurate polygonal meshes for the resulting 3D volumes. It differs from standard 3D voxel rendering by treating blocks as 4D primitives (4-balls or 4-boxes) rather than 3D cubes, allowing the slice geometry to change shape (e.g., from rectangle to hexagon) as the hyperplane rotates or translates through the fourth dimension [S1, unofficial].

## How it works
**Inputs:**
- A 4D voxel world defined as a lattice of 4D primitives.
- A 3D hyperplane defined by its orientation (rotation angles $\theta, \phi$) and position (distance $w$ along the fourth axis).
- A rendering mode flag `P` (boolean).

**Outputs:**
- A 3D mesh representing the visible cross-section of the world.

**Data Structures:**
- **4D Primitive:** Defined by its center coordinates $(x, y, z, w)$, dimensions (for 4-boxes), and radius (for 4-balls).
- **3D Hyperplane:** Defined by normal vector and offset. In this implementation, the camera is fixed in 3D space; the hyperplane moves/rotates relative to the static world.

**Steps:**

1. **Determine Primitive Type:** Identify if the voxel is a 4-ball (boulder) or a 4-box (monolith).

2. **Calculate Intersection Geometry:**
   - **For 4-balls:** Calculate the distance $d$ from the 4D center of the ball to the slicing hyperplane. The cross-section is a 3D sphere with radius $r_{3D} = \sqrt{r^2 - d^2}$, where $r$ is the 4D radius. If $d > r$, the block is not visible.
   - **For 4-boxes:** Intersect the 4D bounding box with the 3D hyperplane. The resulting footprint is a 3D polygon.
     - If the hyperplane rotation involves only one plane, the resulting cells remain four-sided (quadrilaterals). In this specific case, if the rotation angle allows right angles (e.g., 0.45 rad), the blocks can be rendered exactly as cubes [S1, unofficial].
     - If the hyperplane rotation involves two planes (compound turn), the cells become pentagons or hexagons. These require accurate polygonal geometry and cannot be approximated by cubes [S1, unofficial].

3. **Apply Rendering Mode (`P`):**
   - If `P` is false: Render all blocks as cubes. This is an approximation that is exact only when the cut is rotated in one plane and maintains right angles.
   - If `P` is true: Render blocks as their exact 4D cross-sections (polygons). This handles obtuse corners and complex shapes resulting from compound rotations [S1, unofficial].

4. **Greedy Meshing:**
   - Attempt to merge adjacent faces. Note that neighboring cells present polygons at different angles due to the 4D intersection, which traditionally prevents greedy meshing. However, convex set properties under linear maps may allow merging in specific configurations [S1, unofficial].

**Parameters:**
- `P`: Toggle for exact vs. cubic rendering. Default behavior depends on implementation; typically false for performance, true for accuracy.
- `CLIP_TIME_OF_DAY`: Controls the $w$-axis position for capture clips. Default cycles naturally; specific values pin the time [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision Loss in Translation:** Translation along the fourth axis ($w$) is not round-trip invariant due to floating-point precision. Traveling from $w=0$ to $w=9$ and back to $w=0$ results in a difference of $9.77/255$ compared to the original frame. Rotation, however, is byte-identical on round-trip [S1, unofficial].
- **Degenerate Intersections:** If $d > r$ for a 4-ball, the block is empty. For 4-boxes, if the hyperplane misses the box entirely, no geometry is generated.

## Subtopics

### 4D Voxel World Representation
A representation where the world is a four-dimensional lattice. The visible scene is a three-dimensional cross-section defined by a hyperplane. The camera is fixed in 3D space; changing the view involves rotating or translating the hyperplane through the 4D world. It is chosen when the goal is to visualize higher-dimensional spatial relationships without moving the observer. A key pitfall is that translation along the fourth axis ($w$) is not round-trip invariant due to floating-point precision, whereas rotation is byte-identical on round-trip [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dependent on the number of voxels intersected by the hyperplane and the complexity of polygon generation for 4-boxes. Greedy meshing is complicated by varying polygon angles, potentially increasing meshing time.
- **Memory Complexity:** Stores the 4D world lattice and the generated 3D mesh.
- **Strengths:** Provides exact geometric representation of 4D objects in 3D space; allows visualization of "time" or other 4D axes as spatial traversal.
- **Limitations:** Floating-point precision errors in $w$-axis translation; complex polygon generation for compound rotations; greedy meshing is non-trivial due to non-aligned faces.
- **When to use:** When visualizing 4D data or simulating higher-dimensional spatial relationships.
- **When not to use:** When performance is critical and approximate cubic rendering is sufficient, or when round-trip precision in $w$-axis translation is required.

## Background
Originating from a solo C++20 and OpenGL 4.1 voxel engine project by DanielWLiu07, this method demonstrates rendering a 3D cross-section of a 4D world. It challenges the assumption that greedy meshing is impossible for such geometry, leveraging convex set properties. The method highlights the difference between rotational and translational invariance in 4D slicing [S1, unofficial].

## Sources
[S1] DanielWLiu07/voxel-engine, https://github.com/DanielWLiu07/voxel-engine, fetched 2026-10-26 (unofficial)
