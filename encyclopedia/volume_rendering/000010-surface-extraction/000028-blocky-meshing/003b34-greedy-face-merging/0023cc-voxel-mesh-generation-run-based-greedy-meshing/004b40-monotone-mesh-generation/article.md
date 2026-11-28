# Monotone Mesh Generation

Also called: MonotoneMesh, MonotoneMesh::CreateRawMesh

## Overview
Monotone Mesh Generation is a CPU-based algorithm that converts voxel data into a manifold triangle mesh by decomposing the surface into monotone polygons along scan lines and triangulating them [S1, unofficial]. It builds on [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md) by replacing per-voxel face extraction with a sweep-line approach that merges adjacent faces into larger polygons, ensuring manifold geometry without per-voxel vertex duplication [S1, unofficial].

## How it works
**Inputs:**
- `Voxel`: Sparse map of `FIntVector` (x, y, z) to `uint8` (color index).
- `Size`: `FIntVector` representing the bounding box dimensions.
- `Palette`: Array of `FColor` corresponding to color indices.
- `ImportOption`: Configuration struct containing `bImportXYCenter` (boolean).

**Outputs:**
- `OutRawMesh`: Mesh structure containing:
  - `VertexPositions`: `TArray<FVector3f>` of unique vertex coordinates.
  - `WedgeIndices`: `TArray<uint32>` of vertex indices forming triangles (triplets).
  - `WedgeTexCoords[0]`: `TArray<FVector2f>` texture coordinates for each wedge.
  - `WedgeColors`: `TArray<FColor>` color for each wedge.
  - `FaceMaterialIndices`: `TArray<uint32>` material index for each face.
  - `FaceSmoothingMasks`: `TArray<uint32>` smoothing mask for each face.

**Data Structures:**
- `FFace`: Represents a 1D segment on a scan line.
  - `Color`: `int32` (positive for outward-facing, negative for inward-facing/normal flip).
  - `Left`: `float` x-coordinate of the start of the segment.
  - `Right`: `float` x-coordinate of the end of the segment.
- `FPolygon`: Represents a monotone polygon being built.
  - `Color`: `int32`.
  - `Left`: `TArray[FVector2f]` vertices forming the left chain (sorted by Y).
  - `Right`: `TArray[FVector2f]` vertices forming the right chain (sorted by Y).

**Algorithm Steps:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CreateRawMesh(Voxel, Size, ImportOption):
    OutRawMesh = FRawMesh()
    for Dimension in range(3):
        Axis = [Dimension, (Dimension + 1) % 3, (Dimension + 2) % 3]
        Plane = [0, 0, 0]
        for Plane[Axis[2]] in range(Size[Axis[2]]):
            Polygons = CreatePolygons(Voxel, Size, Axis, Plane)
            for Polygon in Polygons:
                WritePolygon(OutRawMesh, Axis, Polygon, Plane)
    if ImportOption.bImportXYCenter:
        Offset = [Size.X * 0.5, Size.Y * 0.5, 0.0]
        for i in range(len(OutRawMesh.VertexPositions)):
            OutRawMesh.VertexPositions[i] = [OutRawMesh.VertexPositions[i][j] - Offset[j] for j in range(3)]
    OutRawMesh.CompactMaterialIndices()
    return OutRawMesh

def CreatePolygons(Voxel, Size, Axis, Plane):
    OutPolygons = []
    for P_y in range(Size[Axis[1]]):
        Faces = CreateFaces(Voxel, Size, Axis, Plane, P_y)
        Frontier = []
        NextFrontier = []
        FrontierIndex = 0
        FaceIndex = 0
        while FrontierIndex < len(Frontier) and FaceIndex < len(Faces):
            poly_idx = Frontier[FrontierIndex]
            poly = OutPolygons[poly_idx]
            face = Faces[FaceIndex]

            left_x = poly.Left[-1][0]
            right_x = poly.Right[-1][0]

            if left_x < face.Right and face.Left < right_x and face.Color == poly.Color:
                poly.Left.append((face.Left, P_y))
                poly.Right.append((face.Right, P_y))
                NextFrontier.append(poly_idx)
                FrontierIndex += 1
                FaceIndex += 1
            else:
                if right_x <= face.Right:
                    poly.CloseOff(P_y, Plane[Axis[2]])
                    FrontierIndex += 1
                if face.Right <= right_x:
                    idx = len(OutPolygons)
                    OutPolygons.append(FPolygon(face.Color, [(face.Left, P_y)], [(face.Right, P_y)]))
                    NextFrontier.append(idx)
                    FaceIndex += 1
        Frontier = NextFrontier
        for idx in Frontier:
            OutPolygons[idx].CloseOff(P_y, Plane[Axis[2]])
    return OutPolygons

def CreateFaces(Voxel, Size, Axis, Plane, P_y):
    OutFaces = []
    D = [0, 0, 0]
    D[Axis[2]] = -1
    prev_color = 0
    for P_x in range(Size[Axis[0]]):
        P = [0, 0, 0]
        P[Axis[0]] = P_x
        P[Axis[1]] = P_y
        P[Axis[2]] = Plane[Axis[2]]
        Back = Voxel.get(tuple(P + D), 0)
        Front = Voxel.get(tuple(P), 0)
        if (Back == 0) == (Front == 0):
            color = 0
        elif Back:
            color = -Back
        else:
            color = Front

        if prev_color != color:
            if prev_color != 0:
                OutFaces[-1].Right = P_x
            if color != 0:
                OutFaces.append(FFace(color, P_x, P_x))
        prev_color = color
    if prev_color != 0:
        OutFaces[-1].Right = P_x
    return OutFaces

def WritePolygon(OutRawMesh, Axis, Polygon, Plane):
    LeftIndex = []
    RightIndex = []
    WriteVertex(OutRawMesh, Axis, Polygon.Left, LeftIndex, Plane)
    WriteVertex(OutRawMesh, Axis, Polygon.Right, RightIndex, Plane)

    ColorIndex = Polygon.Color - 1 if Polygon.Color > 0 else -Polygon.Color - 1
    Flipped = Polygon.Color < 0

    Left = 1
    Right = 1
    LastSide = True
    def CrossProduct(a, b):
        return a[0] * b[1] - b[0] * a[1]
    List = [(LeftIndex[0], Polygon.Left[0]), (RightIndex[0], Polygon.Right[0])]

    while Left < len(Polygon.Left) or Right < len(Polygon.Right):
        if Left == len(Polygon.Left):
            Side = True
        elif Right < len(Polygon.Right):
            Side = Polygon.Left[Left][1] > Polygon.Right[Right][1]
        else:
            Side = False

        if Side:
            Index = RightIndex[Right]
            Vertex = Polygon.Right[Right]
        else:
            Index = LeftIndex[Left]
            Vertex = Polygon.Left[Left]

        if Side != LastSide:
            while len(List) > 1:
                WriteWedge(OutRawMesh, List[0][0], List[1][0], Index, ColorIndex, Flipped == Side)
                List.pop(0)
        else:
            while len(List) > 1:
                Normal = CrossProduct([List[-1][1][0]-Vertex[0], List[-1][1][1]-Vertex[1]], [List[-2][1][0]-Vertex[0], List[-2][1][1]-Vertex[1]])
                if Side == (Normal > 0):
                    break
                if Normal != 0:
                    WriteWedge(OutRawMesh, List[-1][0], List[-2][0], Index, ColorIndex, Flipped == Side)
                    List.pop()
        List.append((Index, Vertex))
        if Side:
            Right += 1
        else:
            Left += 1
        LastSide = Side

def WriteVertex(OutRawMesh, Axis, Chain, IndexArray, Plane):
    for pt in Chain:
        Vertex = list(Plane)
        Vertex[Axis[1]] += pt[0]
        Vertex[Axis[2]] += pt[1]
        idx = OutRawMesh.VertexPositions.AddUnique(Vertex)
        IndexArray.append(idx)

def WriteWedge(OutRawMesh, Index1, Index2, Index3, ColorIndex, Flipped):
    OutRawMesh.WedgeIndices.Add(Index1 if not Flipped else Index2)
    OutRawMesh.WedgeIndices.Add(Index2 if not Flipped else Index1)
    OutRawMesh.WedgeIndices.Add(Index3)
    TexCoord = (((ColorIndex + 0.5) / 256.0), 0.5)
    OutRawMesh.WedgeTexCoords[0].Add(TexCoord)
    OutRawMesh.WedgeTexCoords[0].Add(TexCoord)
    OutRawMesh.WedgeTexCoords[0].Add(TexCoord)
    OutRawMesh.FaceMaterialIndices.Add(0)
    OutRawMesh.FaceSmoothingMasks.Add(0)
```

**Parameters:**
- `ImportOption->bImportXYCenter`: Boolean. If true, centers the mesh in XY by subtracting `(Size.X/2, Size.Y/2, 0)` from all vertices [S1, unofficial].
- `ColorIndex`: Used in `WriteWedge` to generate texture coordinates. Formula: `((double)ColorIndex + 0.5) / 256.0` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: The algorithm relies on exact comparisons of integer voxel coordinates for face detection [S1, unofficial].
- **Boundaries**: Loop bounds follow `Voxel.Size` to iterate over valid planes and scan lines [S1, unofficial].
- **Degenerate Input**: Empty voxel sets result in empty meshes; single-voxel meshes are handled correctly by the polygon merging logic [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Not given in the sources.
- **Strengths**: Produces manifold geometry without per-voxel vertex duplication [S1, unofficial].
- **Limitations**: Not given in the sources.
- **When to Use**: Not given in the sources.
- **When Not to Use**: Not given in the sources.

## Background
Monotone Mesh Generation is based on the method described at https://0fps.net/2012/07/07/meshing-minecraft-part-2/ [S1, unofficial]. It is implemented in the `VOX4U` plugin for Unreal Engine by mik14a [S1, unofficial]. The method is used to import MagicaVoxel `.vox` files into Unreal Engine as optimized raw meshes [S1, unofficial].

## Sources
[S1] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
