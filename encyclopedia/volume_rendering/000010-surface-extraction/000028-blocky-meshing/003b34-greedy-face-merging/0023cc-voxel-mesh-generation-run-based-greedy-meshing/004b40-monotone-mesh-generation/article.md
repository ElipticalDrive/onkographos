# Monotone Mesh Generation

Also called: MonotoneMesh, MonotoneMesh::CreateRawMesh

## Overview
Monotone Mesh Generation is a CPU-based algorithm that converts voxel data into a manifold triangle mesh by decomposing the surface into monotone polygons along scan lines and triangulating them [S1, unofficial]. It builds on [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md) by replacing per-voxel face extraction with a sweep-line approach that merges adjacent faces into larger polygons, ensuring manifold geometry without per-voxel vertex duplication [S1, unofficial]. Unlike the basic greedy approach which optimizes for speed over minimal triangle count, this method produces cleaner, lower-poly geometry by creating larger planar patches [S1, unofficial].

## How it works
**Inputs:**
- `Voxel`: A sparse map of `FIntVector` (x, y, z) to `uint8` (color index).
- `Size`: `FIntVector` representing the bounding box dimensions of the voxel data.
- `Palette`: Array of `FColor` corresponding to color indices.
- `ImportOption`: Configuration struct containing `bImportXYCenter` (boolean).

**Outputs:**
- `OutRawMesh`: A mesh structure containing:
  - `VertexPositions`: `TArray<FVector3f>` of unique vertex coordinates.
  - `WedgeIndices`: `TArray<uint32>` of vertex indices forming triangles (triplets).
  - `WedgeTexCoords[0]`: `TArray<FVector2f>` texture coordinates for each wedge.
  - `WedgeColors`: `TArray<FColor>` color for each wedge.
  - `FaceMaterialIndices`: `TArray<uint32>` material index for each face (polygon).
  - `FaceSmoothingMasks`: `TArray<uint32>` smoothing mask for each face.

**Data Structures:**
- `FFace`: Represents a 1D segment on a scan line.
  - `Color`: `int32` (positive for outward-facing, negative for inward-facing/normal flip).
  - `Left`: `float` x-coordinate of the start of the segment.
  - `Right`: `float` x-coordinate of the end of the segment.
- `FPolygon`: Represents a monotone polygon being built.
  - `Color`: `int32`.
  - `Left`: `TArray<FVector2f>` vertices forming the left chain (sorted by Y).
  - `Right`: `TArray<FVector2f>` vertices forming the right chain (sorted by Y).

**Algorithm Steps:**

1. **Initialization**: Initialize `OutRawMesh` arrays.
2. **Sweep Loop**: Iterate `Dimension` from 0 to 2 (representing X, Y, Z axes).
   - Define `Axis` vector: `Axis[0] = Dimension`, `Axis[1] = (Dimension + 1) % 3`, `Axis[2] = (Dimension + 2) % 3`.
   - This defines the sweep direction (`Axis[0]`), the scan line direction (`Axis[1]`), and the plane normal direction (`Axis[2]`).
   - Iterate `Plane[Axis[2]]` from 0 to `Voxel.Size[Axis[2]]`.
     - Call `CreatePolygons` to generate a list of `FPolygon` objects for this plane slice.
     - For each `Polygon` in the list, call `WritePolygon` to triangulate and write to `OutRawMesh`.
3. **Centering**: If `ImportOption->bImportXYCenter` is true, calculate `Offset = (Size.X * 0.5, Size.Y * 0.5, 0)`. Subtract `Offset` from all `VertexPositions`.
4. **Finalization**: Call `OutRawMesh.CompactMaterialIndices()`.

**Sub-algorithm: CreatePolygons**
This function generates monotone polygons for a specific plane slice.

1. Initialize `OutPolygons` (list of `FPolygon`).
2. Iterate `P[Axis[1]]` (scan line position) from 0 to `Voxel.Size[Axis[1]]`.
   - Call `CreateFaces` to get a list of `FFace` objects for this scan line.
   - Initialize `Frontier` (list of indices into `OutPolygons`) and `NextFrontier`.
   - Initialize `FrontierIndex = 0`, `FaceIndex = 0`.
   - **Merge Loop**: While `FrontierIndex < Frontier.Num()` AND `FaceIndex < Faces.Num()`:
     - Get `Polygon = OutPolygons[Frontier[FrontierIndex]]`.
     - Get `CurrentFace = Faces[FaceIndex]`.
     - Let `Left.X` be the last element of `Polygon.Left`'s X-coordinate (or start if empty).
     - Let `Right.X` be the last element of `Polygon.Right`'s X-coordinate.
     - **Condition**: If `Left.X < CurrentFace.Right` AND `CurrentFace.Left < Right.X` AND `CurrentFace.Color == Polygon.Color`:
       - **Merge**: Add `CurrentFace.Left` and `CurrentFace.Right` to `Polygon.Left` and `Polygon.Right` respectively. The Y-coordinate for these points is `P[Axis[1]]`. The Z-coordinate is `Plane[Axis[2]]`.
       - Add `Frontier[FrontierIndex]` to `NextFrontier`.
       - Increment `FrontierIndex` and `FaceIndex`.
     - **Else**:
       - If `Right.X <= CurrentFace.Right`:
         - **Close Polygon**: Call `Polygon.CloseOff(P[Axis[1]], Plane[Axis[2]])`. This finalizes the polygon's last vertices.
         - Increment `FrontierIndex`.
       - If `CurrentFace.Right <= Right.X`:
         - **Start New Polygon**: Add `OutPolygons.Num()` to `NextFrontier`. Create new `FPolygon` with `CurrentFace.Color`, `CurrentFace.Left`, `CurrentFace.Right`, and initial vertices at `P[Axis[1]], Plane[Axis[2]]`. Add to `OutPolygons`.
         - Increment `FaceIndex`.
   - **Remaining Frontiers**: If `FrontierIndex < Frontier.Num()`, close all remaining polygons in `Frontier`.
   - **Remaining Faces**: If `FaceIndex < Faces.Num()`, create new polygons for remaining faces.
   - Update `Frontier = NextFrontier`.
3. **Final Close**: After the Y loop, close any remaining polygons in `Frontier`.

**Sub-algorithm: CreateFaces**
This function extracts 1D segments (faces) from the voxel data along a scan line.

1. Initialize `OutFaces` (list of `FFace`).
2. Initialize `D` vector: `D[Axis[0]] = 0`, `D[Axis[1]] = 0`, `D[Axis[2]] = -1`.
3. Initialize `PreviousColor = 0`.
4. Iterate `P[Axis[0]]` from 0 to `Voxel.Size[Axis[0]]`.
   - Calculate `BackPos = P + D`.
   - Get `Back = Voxel.FindRef(BackPos)` (returns color or 0 if empty).
   - Get `Front = Voxel.FindRef(P)`.
   - Calculate `Color`:
     - If `!Back` and `!Front` (both empty): `Color = 0`.
     - Else if `Back` (back is solid): `Color = -Back` (inward facing).
     - Else (`Front` is solid): `Color = Front` (outward facing).
   - **Edge Detection**:
     - If `PreviousColor != Color`:
       - If `PreviousColor != 0`: Set `OutFaces.Last().Right = P[Axis[0]]`.
       - If `Color != 0`: Add new `FFace(Color, P[Axis[0]], P[Axis[0]])` to `OutFaces`.
     - Set `PreviousColor = Color`.
5. **Finalize**: If `PreviousColor != 0`, set `OutFaces.Last().Right = P[Axis[0]]`.

**Sub-algorithm: WritePolygon**
This function triangulates a monotone polygon and writes triangles to the mesh.

1. Initialize `LeftIndex` and `RightIndex` arrays.
2. Call `WriteVertex` for `Polygon.Left` to populate `LeftIndex` and `OutRawMesh.VertexPositions`.
3. Call `WriteVertex` for `Polygon.Right` to populate `RightIndex` and `OutRawMesh.VertexPositions`.
4. Calculate `ColorIndex`: `0 < Polygon.Color ? Polygon.Color - 1 : -Polygon.Color - 1`.
5. Calculate `Flipped`: `Polygon.Color < 0`.
6. Initialize `Left = 1`, `Right = 1`, `LastSide = true`.
7. Define `CrossProduct(a, b) = a.X * b.Y - b.X * a.Y`.
8. Initialize `List` with `TPair(LeftIndex[0], Polygon.Left[0])` and `TPair(RightIndex[0], Polygon.Right[0])`.
9. **Triangulation Loop**: While `Left < Polygon.Left.Num()` OR `Right < Polygon.Right.Num()`:
   - Determine `Side`:
     - If `Left == Polygon.Left.Num()`: `Side = true` (Right).
     - Else if `Right != Polygon.Right.Num()`: `Side = (Polygon.Left[Left].Y > Polygon.Right[Right].Y)`.
     - Else: `Side = false` (Left).
   - Get `Index` and `Vertex` based on `Side` (`RightIndex[Right]/Polygon.Right[Right]` if Side, else `LeftIndex[Left]/Polygon.Left[Left]`).
   - **Wedge Generation**:
     - If `Side != LastSide`:
       - While `List.Num() > 1`:
         - Write wedge using `List[0]`, `List[1]`, `Index`, `Color`, and `Flipped == Side`.
         - Remove `List[0]`.
     - Else (`Side == LastSide`):
       - While `List.Num() > 1`:
         - Calculate `Normal = CrossProduct(List.Last().Value - Vertex, List[List.Num()-2].Value - Vertex)`.
         - If `Side == (Normal > 0)`: Break.
         - If `Normal != 0`: Write wedge using `List.Last()`, `List[List.Num()-2]`, `Index`, `Color`, `Flipped == Side`.
         - Remove `List.Last()`.
   - Add `TPair(Index, Vertex)` to `List`.
   - Increment `Right` if `Side`, else `Left`.
   - Set `LastSide = Side`.

**Sub-algorithm: WriteVertex**
1. Iterate `i` from 0 to `Polygon.Left.Num()`.
   - Create `Vertex` from `Polygon.Left[i]` mapped to `Axis` coordinates.
   - Add `Vertex` to `OutRawMesh.VertexPositions` using `AddUnique`.
   - Store index in `OutLeftIndex[i]`.
2. Iterate `i` from 0 to `Polygon.Right.Num()`.
   - Create `Vertex` from `Polygon.Right[i]` mapped to `Axis` coordinates.
   - Add `Vertex` to `OutRawMesh.VertexPositions` using `AddUnique`.
   - Store index in `OutRightIndex[i]`.

**Sub-algorithm: WriteWedge**
1. Add three indices to `OutRawMesh.WedgeIndices`: `Index1` if `Face` else `Index2`, `Index2` if `Face` else `Index1`, `Index3`.
2. Add three identical texture coordinates to `OutRawMesh.WedgeTexCoords[0]`: `((double)ColorIndex + 0.5) / 256.0`, `0.5`.
3. Add `0` to `OutRawMesh.FaceMaterialIndices` and `OutRawMesh.FaceSmoothingMasks`.

**Parameters:**
- `ImportOption->bImportXYCenter`: Boolean. If true, centers the mesh in XY by subtracting `(Size.X/2, Size.Y/2, 0)` from all vertices [S1, unofficial].
- `ColorIndex`: Used in `WriteWedge` to generate texture coordinates. Formula: `((double)ColorIndex + 0.5) / 256.0` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: The algorithm relies on exact comparisons of integer voxel coordinates. Floating point precision issues are minimized by using integer arithmetic for voxel access and only converting to float for vertex positions [S1, unofficial].
- **Boundaries**: The loop bounds must strictly follow `Voxel.Size` to avoid out-of-bounds access [S1, unofficial].
- **Degenerate Input**: Empty voxel sets result in empty meshes. Single-voxel meshes are handled correctly by the polygon merging logic [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources. The algorithm involves multiple nested loops over the voxel dimensions, with polygon merging and triangulation.
- **Memory Complexity**: Not given in the sources. Depends on the number of polygons and vertices generated.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Not given in the sources.
- **Strengths**: Produces manifold geometry without per-voxel duplication. Generates larger, cleaner polygons compared to basic greedy meshing.
- **Limitations**: More complex to implement than basic greedy meshing. May not produce the absolute minimal triangle count compared to more advanced algorithms.
- **When to Use**: When manifold geometry and cleaner meshes are prioritized over raw generation speed. Suitable for voxel worlds where visual quality is important.
- **When Not to Use**: When maximum generation speed is required and per-voxel duplication is acceptable.

## Background
Monotone Mesh Generation is based on the method described at https://0fps.net/2012/07/07/meshing-minecraft-part-2/ [S1, unofficial]. It is implemented in the `VOX4U` plugin for Unreal Engine by mik14a [S1, unofficial]. The method is used to import MagicaVoxel `.vox` files into Unreal Engine as optimized raw meshes [S1, unofficial].

## Sources
[S1] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
