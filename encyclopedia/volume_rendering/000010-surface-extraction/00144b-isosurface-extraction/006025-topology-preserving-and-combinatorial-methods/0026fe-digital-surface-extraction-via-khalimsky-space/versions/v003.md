# Digital Surface Extraction via Khalimsky Space

Also called: Combinatorial manifold surface extraction, Digital Surface Extraction via Khalimsky Space and Surfel Adjacency, Digital boundary scanning, Dual surface to the digital boundary, Marching-cube like surface extracted using the combinatorial manifold structure of digital surfaces

## Overview
Digital Surface Extraction via Khalimsky Space extracts a boundary surface from a digital image by constructing a Khalimsky space, defining a digital set based on intensity thresholds, and scanning the space to identify surfels (surface elements) using a specific adjacency rule (interior or exterior) [S1][S2, unofficial]. It differs from [Isosurface extraction](30-isosurface-extraction.md) methods like Marching Cubes by relying on combinatorial topology and digital geometry rather than geometric interpolation within grid cells, producing a dual surface to the digital boundary [S1][S2, unofficial].

## How it works
**Inputs:**
- A volumetric image (grid of scalar values) [S1][S2, unofficial].
- `minThreshold` (unsigned int): Minimum voxel intensity for inclusion [S2, unofficial].
- `maxThreshold` (unsigned int): Maximum voxel intensity for inclusion [S2, unofficial].
- `intAdjacency` (bool): Connectivity rule (true for interior, false for exterior) [S2, unofficial].

**Outputs:**
- A triangulated or polygonal mesh representing the dual surface to the digital boundary [S2, unofficial].

**Data Structures:**
- `DigitalSet`: A set of voxels belonging to the shape. A voxel $v$ belongs to the shape if $minThreshold \le I(v) \le maxThreshold$ [S2, unofficial] (note: [S1] uses $minThreshold < I(v) \le maxThreshold$).
- `KSpace`: The Khalimsky space constructed from the image domain's lower and upper bounds [S1][S2, unofficial].
- `SurfelAdjacency`: Defines connectivity between surfels [S1][S2, unofficial].
- `DigitalSurface`: A container of surfels forming the boundary [S1][S2, unofficial].
- `CellEmbedder`: Maps digital cells to real coordinates using an offset [S1][S2, unofficial].
- `TriangulatedSurface` / `PolygonalSurface`: The final mesh representation [S1][S2, unofficial].

**Steps:**

1.  **Read Image:** Load the volume file into an image structure.
    ```python
    volume = load_volume(volume_path)  # [G] Reads volume file into 3D array
    lower, upper = volume.bounds  # [G] Grid domain bounds
    ```
2.  **Create Digital Set:** Initialize a digital set and populate it with voxels satisfying the threshold criteria.
    ```python
    digital_set = set()
    for v in volume.voxels:
        if min_threshold <= volume[v] <= max_threshold:  # [S2, unofficial] (Note: [S1] uses < min_threshold)
            digital_set.add(v)
    ```
3.  **Initialize Khalimsky Space:** Construct the Khalimsky space from the image domain.
    ```python
    ks = KSpace()
    ks.init(lower, upper, True)  # [S1][S2, unofficial]
    ```
4.  **Define Surfel Adjacency:** Initialize the adjacency object.
    ```python
    surf_adj = SurfelAdjacency(ks.dimension, int_adjacency)  # [S1][S2, unofficial]
    ```
5.  **Extract Boundary Surfels:** Scan the space to identify the boundary surfels.
    ```python
    set_of_surfels = SetOfSurfels(ks, surf_adj)
    Surfaces.sMakeBoundary(set_of_surfels.surfel_set(), ks, digital_set, lower, upper)  # [S1][S2, unofficial]
    dig_surf = DigitalSurface(set_of_surfels)
    ```
6.  **Initialize Embedder:** Create an embedder to map digital cells to real coordinates. An offset of `minThreshold + 0.5` is added to avoid the isosurface passing exactly through voxel centers, which causes geometric ambiguity [S1][S2, unofficial].
    ```python
    trivial_embedder = CanonicEmbedder()  # [G] Canonical injection Z^n -> R^n
    cell_embedder = ImageLinearCellEmbedder(ks, volume, trivial_embedder)
    iso_level = min_threshold + 0.5  # [S1][S2, unofficial] Offset avoids voxel center intersection
    cell_embedder.init(ks, volume, trivial_embedder, iso_level)
    ```
7.  **Convert to Mesh:** Convert the digital surface to a dual polygonal or triangulated surface.
    ```python
    vmap = {}  # [G] Vertex index mapping
    poly_mesh = PolygonalSurface()
    MeshHelpers.digital_surface2_dual_polygonal_surface(dig_surf, cell_embedder, poly_mesh, vmap)  # [S2, unofficial]
    ```
8.  **Finalize Mesh:** Convert the polygonal/triangulated surface to a viewable mesh format.
    ```python
    view_mesh = Mesh()
    MeshHelpers.polygonal_surface2_mesh(poly_mesh, view_mesh)  # [S2, unofficial]
    return view_mesh
    ```

**Parameters:**
- `minThreshold`: Typical usage is passed as a command-line argument [S2, unofficial].
- `maxThreshold`: Typical usage is passed as a command-line argument [S2, unofficial].
- `intAdjacency`: `0` (true) indicates interior adjacency; `1` (false) indicates exterior adjacency [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Binary Volumes:** If the isosurface level equals an integer voxel value (e.g., `minThreshold` without offset), the surface may pass exactly through voxel centers, causing geometric ambiguity or degeneracy [S1][S2, unofficial]. The `+0.5` offset handles this [S1][S2, unofficial].
- **Space Construction:** Failure to initialize the Khalimsky space correctly will abort the process [S1][S2, unofficial].

## Subtopics

### Dual Triangulated Surface Generation
A variant that converts the digital surface (set of surfels) directly into a dual triangulated mesh (`TriangulatedSurface`) rather than a polygonal surface [S1]. It uses the same Khalimsky space and surfel extraction but employs `MeshHelpers::digitalSurface2DualTriangulatedSurface` and `ImageLinearCellEmbedder` with a `CanonicEmbedder` to map integer coordinates to real coordinates [S1].

## Operating characteristics
- **Time Complexity:** The boundary extraction step (`sMakeBoundary`) dominates the runtime. In the provided example, extracting the boundary for a lobster volume took 2757.2 ms, while mesh conversion took 2380.92 ms [S2, unofficial].
- **Memory Complexity:** Depends on the number of surfels and the size of the Khalimsky space. The example resulted in 160,660 surfels [S1][S2, unofficial].
- **Strengths:** Guarantees topological correctness (manifold surface) by construction [S1][S2, unofficial].
- **Limitations:** Requires careful thresholding and adjacency selection. Not suitable for non-grid data without discretization [S1][S2, unofficial].

## Background
The method originates from the DGtal library, which implements digital geometry and topology tools [S1][S2, unofficial]. It is based on the theory of Khalimsky spaces, which provide a combinatorial framework for defining digital surfaces and their adjacency properties [S1][S2, unofficial].

## Sources
[S1] shapes/viewMarchingCubes.cpp, https://www.dgtal.org/doc/1.4/shapes_2viewMarchingCubes_8cpp-example.html, fetched 2026-10-28
[S2] shapes/viewPolygonalMarchingCubes.cpp, https://www.dgtal.org/doc/1.4/shapes_2viewPolygonalMarchingCubes_8cpp-example.html, fetched 2026-10-28 (unofficial)
