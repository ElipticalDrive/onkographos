# Digital Surface Extraction via Khalimsky Space

Also called: Combinatorial manifold surface extraction, Digital Surface Extraction via Khalimsky Space and Surfel Adjacency, Digital boundary scanning, Dual surface to the digital boundary, Marching-cube like surface extracted using the combinatorial manifold structure of digital surfaces

## Overview
Digital Surface Extraction via Khalimsky Space extracts a boundary surface from a digital image by constructing a Khalimsky space, defining a digital set based on intensity thresholds, and scanning the space to identify surfels (surface elements) using a specific adjacency rule (interior or exterior) [S1][S2, unofficial]. It differs from [Isosurface extraction](30-isosurface-extraction.md) methods like Marching Cubes by relying on combinatorial topology and digital geometry rather than geometric interpolation within grid cells, ensuring a topologically correct manifold surface [S1][S2, unofficial].

## How it works
**Inputs:**
- A volumetric image (grid of scalar values).
- `minThreshold` (unsigned int): Minimum voxel intensity for inclusion.
- `maxThreshold` (unsigned int): Maximum voxel intensity for inclusion.
- `intAdjacency` (bool): Connectivity rule (true for interior, false for exterior).

**Outputs:**
- A triangulated or polygonal mesh representing the dual surface to the digital boundary.

**Data Structures:**
- `DigitalSet`: A set of voxels belonging to the shape. A voxel $v$ belongs to the shape if $minThreshold < I(v) \le maxThreshold$ [S1] or $minThreshold \le I(v) \le maxThreshold$ [S2, unofficial].
- `KSpace`: The Khalimsky space constructed from the image domain's lower and upper bounds.
- `SurfelAdjacency`: Defines connectivity between surfels.
- `DigitalSurface`: A container of surfels forming the boundary.
- `CellEmbedder`: Maps digital cells to real coordinates using an offset.
- `TriangulatedSurface` / `PolygonalSurface`: The final mesh representation.

**Steps:**

1.  **Read Image:** Load the volume file into an image structure.
    ```python
    image = VolReader.importVol(inputFilename)
    ```
2.  **Create Digital Set:** Initialize a digital set and populate it with voxels satisfying the threshold criteria.
    ```python
    # Using the condition minThreshold <= I(v) <= maxThreshold as per [S2, unofficial]
    # Note: [S1] uses minThreshold < I(v) <= maxThreshold. Implementation should follow the specific library version.
    set3d = DigitalSet(image.domain())
    SetFromImage.append(set3d, image, minThreshold, maxThreshold)
    ```
3.  **Initialize Khalimsky Space:** Construct the Khalimsky space from the image domain.
    ```python
    ks = KSpace()
    space_ok = ks.init(image.domain().lowerBound(), image.domain().upperBound(), True)
    if not space_ok:
        raise Error("Khalimsky space construction failed")
    ```
4.  **Define Surfel Adjacency:** Initialize the adjacency object.
    ```python
    # intAdjacency=True corresponds to interior adjacency (0 in command line args)
    surfAdj = SurfelAdjacency(KSpace.dimension, intAdjacency)
    ```
5.  **Extract Boundary Surfels:** Scan the space to identify the boundary surfels.
    ```python
    theSetOfSurfels = SetOfSurfels(ks, surfAdj)
    Surfaces.sMakeBoundary(
        theSetOfSurfels.surfelSet(),
        ks,
        set3d,
        image.domain().lowerBound(),
        image.domain().upperBound()
    )
    digSurf = DigitalSurface(theSetOfSurfels)
    ```
6.  **Initialize Embedder:** Create an embedder to map digital cells to real coordinates. An offset of `minThreshold + 0.5` is added to avoid the isosurface passing exactly through voxel centers, which causes geometric ambiguity [S1][S2, unofficial].
    ```python
    trivialEmbedder = CanonicEmbedder(Space)
    cellEmbedder = ImageLinearCellEmbedder(ks, image, trivialEmbedder)
    # Offset to avoid voxel center intersection
    isoLevel = minThreshold + 0.5
    cellEmbedder.init(ks, image, trivialEmbedder, isoLevel)
    ```
7.  **Convert to Mesh:** Convert the digital surface to a dual polygonal or triangulated surface.
    ```python
    # For Polygonal Surface
    polymesh = PolygonalSurface()
    vmap = VertexMap()
    MeshHelpers.digitalSurface2DualPolygonalSurface(digSurf, cellEmbedder, polymesh, vmap)
    
    # For Triangulated Surface (Variant)
    # trimesh = TriangulatedSurface()
    # MeshHelpers.digitalSurface2DualTriangulatedSurface(digSurf, cellEmbedder, trimesh, vmap)
    ```
8.  **Finalize Mesh:** Convert the polygonal/triangulated surface to a viewable mesh format.
    ```python
    viewmesh = Mesh()
    MeshHelpers.polygonalSurface2Mesh(polymesh, viewmesh)
    # Or for triangulated:
    # MeshHelpers.triangulatedSurface2Mesh(trimesh, viewmesh)
    ```

**Parameters:**
- `minThreshold`: Typical usage is passed as a command-line argument.
- `maxThreshold`: Typical usage is passed as a command-line argument. Default is often 255 for 8-bit images [S1].
- `intAdjacency`: `0` (true) indicates interior adjacency; `1` (false) indicates exterior adjacency [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Binary Volumes:** If the isosurface level equals an integer voxel value (e.g., `minThreshold` without offset), the surface may pass exactly through voxel centers, causing geometric ambiguity or degeneracy [S1][S2, unofficial]. The `+0.5` offset handles this [S1][S2, unofficial].
- **Space Construction:** Failure to initialize the Khalimsky space correctly will abort the process [S1][S2, unofficial].

## Subtopics

### Dual Triangulated Surface Generation
A variant that converts the digital surface (set of surfels) directly into a dual triangulated mesh (`TriangulatedSurface`) rather than a polygonal surface [S1]. It uses the same Khalimsky space and surfel extraction but employs `MeshHelpers::digitalSurface2DualTriangulatedSurface` and `ImageLinearCellEmbedder` with a `CanonicEmbedder` to map integer coordinates to real coordinates [S1]. It is chosen when a strictly triangulated output is required for downstream rendering or analysis [S1].

## Operating characteristics
- **Time Complexity:** The boundary extraction step (`sMakeBoundary`) dominates the runtime. In the provided example, extracting the boundary for a lobster volume took 2757.2 ms, while mesh conversion took 2380.92 ms [S2, unofficial].
- **Memory Complexity:** Depends on the number of surfels and the size of the Khalimsky space. The example resulted in 160,660 surfels [S1][S2, unofficial].
- **Performance:** The method is robust for digital images but may be slower than geometric methods like Marching Cubes for large volumes due to the overhead of combinatorial topology operations [S1][S2, unofficial].
- **Strengths:** Guarantees topological correctness (manifold surface) by construction [S1][S2, unofficial].
- **Limitations:** Requires careful thresholding and adjacency selection. Not suitable for non-grid data without discretization [S1][S2, unofficial].

## Background
The method originates from the DGtal library, which implements digital geometry and topology tools [S1][S2, unofficial]. It is based on the theory of Khalimsky spaces, which provide a combinatorial framework for defining digital surfaces and their adjacency properties [S1][S2, unofficial]. The approach was developed to provide a rigorous topological alternative to geometric isosurface extraction methods [S1][S2, unofficial].

## Sources
[S1] shapes/viewMarchingCubes.cpp, https://www.dgtal.org/doc/1.4/shapes_2viewMarchingCubes_8cpp-example.html, fetched 2026-10-28
[S2] shapes/viewPolygonalMarchingCubes.cpp, https://www.dgtal.org/doc/1.4/shapes_2viewPolygonalMarchingCubes_8cpp-example.html, fetched 2026-10-28 (unofficial)
