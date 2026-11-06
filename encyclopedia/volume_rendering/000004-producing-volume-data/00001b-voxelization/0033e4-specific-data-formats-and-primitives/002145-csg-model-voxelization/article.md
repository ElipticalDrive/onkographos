# CSG Model Voxelization

Also called: Boolean CSG voxelization

## Overview
CSG Model Voxelization is a method for converting Constructive Solid Geometry (CSG) models into voxel grids by voxelizing individual CSG nodes into intermediate "worksheets" and performing boolean operations in image space using GPU blending [S1, unofficial]. It differs from its parent method, [Specific Data Formats and Primitives](7148-specific-data-formats-and-primitives.md), by leveraging the density computation inherent in solid voxelization to recover intersection volumes rapidly, making it particularly suitable for collision detection and haptic feedback [S1, unofficial]. Unlike [SDF-based Voxelization](6595-sdf-based-voxelization.md), which relies on signed distance fields for smooth boundaries, this method operates directly on discrete voxel grids to achieve real-time performance with low dependence on input geometry complexity [S1, unofficial].

## How it works
**Inputs:** A CSG tree representing the model, where each leaf node is a geometric primitive (mesh or analytic primitive).
**Outputs:** A 3D voxel grid (worksheet) representing the final CSG result.

**Data Structures:**
*   **Worksheet:** A 2D texture array or volume texture representing a slice or the full voxel grid of a CSG node. The exact layout is not specified in the sources, but it functions as a 2D texture for GPU blending operations [S1, unofficial].
*   **CSG Tree:** A hierarchical structure of boolean operations (Union, Intersection, Difference) and leaf primitives.

**Steps:**
1.  **Voxelize CSG Nodes:** Each leaf node of the CSG tree is voxelized individually into its respective worksheet. This involves processing the geometry of the primitive and transcoding its positions into the worksheet coordinate system [S1, unofficial]. For parametric surfaces, vertex shader texture coordinates in the parametric domain are used to calculate positions in 3D space, which are then transcoded to the worksheet immediately [S2, unofficial].
2.  **Perform CSG Constructions:** The boolean operations are performed from the bottom of the CSG tree to the top. This is done automatically in image space by applying boolean operations on the 2D textures (worksheets) defined by GPU blending operations [S1, unofficial]. The GPU's blending capabilities are used to execute the boolean logic (e.g., union, intersection) on the voxel data [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_csg_model(csg_tree):
    """
    Voxelize a CSG model by voxelizing nodes and performing boolean operations
    in image space using GPU blending.
    """
    # Step 1: Voxelize leaf nodes into worksheets
    worksheets = {}
    for node in csg_tree.leaves:
        # Voxelize the primitive geometry into a 2D texture/volume
        # For parametric surfaces, use texture coordinates to map to 3D space
        worksheet = voxelize_primitive(node.geometry)
        worksheets[node.id] = worksheet

    # Step 2: Perform CSG constructions bottom-up
    # This step relies on GPU blending operations for boolean logic
    # The exact implementation of boolean operations via blending is not detailed
    # in the sources, but it is performed automatically in image space [S1, unofficial]
    
    # Placeholder for the GPU-based boolean operation logic
    # The sources state that boolean operations on 2D textures are well-defined
    # by blending operations in GPU [S1, unofficial]
    final_worksheet = perform_csg_operations(csg_tree, worksheets)
    
    return final_worksheet

def voxelize_primitive(geometry):
    """
    Voxelize a single geometric primitive into a worksheet.
    """
    # Process each point as a vertex in a vertex shader
    # Use texture coordinates in parametric domain to calculate 3D positions
    # Transcode positions to the worksheet immediately [S2, unofficial]
    worksheet = create_empty_worksheet()
    for vertex in geometry.vertices:
        # Calculate 3D position from parametric coordinates
        pos_3d = calculate_3d_position(vertex.parametric_coords)
        # Transcode to worksheet coordinates
        worksheet_coords = transcoding_to_worksheet(pos_3d)
        # Set voxel value in worksheet
        set_voxel(worksheet, worksheet_coords, 1.0) # Assuming binary occupancy
    return worksheet

def perform_csg_operations(csg_tree, worksheets):
    """
    Perform boolean operations on worksheets bottom-up.
    Note: The specific blending operations for boolean logic are not detailed
    in the sources [S1, unofficial].
    """
    # This function would traverse the CSG tree and apply boolean operations
    # using GPU blending on the worksheets.
    # Since the sources do not provide the exact blending equations or shader code,
    # this is a high-level representation.
    pass
```

**Parameters:**
*   **Resolution:** Not given in the sources.
*   **Blending Mode:** Not given in the sources, but GPU blending operations are used for boolean logic [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### [CSG and Inter-object Intersection](4703-csg-and-inter-object-intersection.md)
CSG and Inter-object Intersection performs general Constructive Solid Geometry (CSG) operations on voxelized objects by rendering individual object slicemaps over one another using GPU blending. It differs from its parent method, [CSG Model Voxelization](2497-csg-model-voxelization.md), by explicitly leveraging the density computation inherent in solid voxelization to recover intersection volumes rapidly, making it particularly suitable for collision detection and haptic feedback. Unlike [SDF-based Voxelization](6595-sdf-based-voxelization.md), which relies on signed distance fields for smooth boundaries, this method operates directly on discrete voxel grids to achieve real-time performance with low dependence on input geometry complexity.

### [SDF-based Voxelization](6595-sdf-based-voxelization.md)
SDF-based Voxelization is a method to visualize any Constructive Solid Geometry (CSG) mesh or analytic primitive as a Minecraft-style voxel chunk pile by determining world-space bounding boxes, sampling a 3D grid, and querying Signed Distance Fields (SDF) to determine occupancy. It differs from GPU-based rasterization voxelization by relying on CPU-side SDF queries and storing results in a packed bitmap for later rendering as analytic box primitives, enabling A/B comparison with the original mesh without shader changes.

### [Sequential CSG](7629-sequential-csg.md)
Sequential CSG is a CPU-based algorithm for performing boolean operations (Union, Intersection, Difference) on two voxel grids by iterating through them word-by-word using word-level parallelism. It differs from GPU-based CSG approaches by operating entirely on the host, processing linear memory blocks of voxels rather than relying on parallel thread execution or image-space blending. It serves as a baseline or fallback implementation for Constructive Solid Geometry (CSG) model voxelization when hardware acceleration is unavailable or when debugging sequential logic is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Real-time performance with low dependence on input geometry complexity [S1, unofficial]. Rapid voxelization enables simple transparent illustration [S2, unofficial].
*   **Limitations:** The method relies on GPU blending operations, which may limit its applicability on hardware without robust blending support. The exact boolean operations via blending are not detailed, which may make implementation difficult [S1, unofficial].
*   **When to Use:** Suitable for collision detection and haptic feedback [S1, unofficial].
*   **When Not to Use:** Not suitable when smooth boundaries are required (use [SDF-based Voxelization](6595-sdf-based-voxelization.md) instead) [S1, unofficial].

## Background
The method was introduced in the paper "Real-time Voxelization for Complex Models" [S1, unofficial]. The paper discusses the voxelization of CSG models as straightforward, leveraging the GPU's ability to perform boolean operations on 2D textures via blending [S2, unofficial]. The method is part of a broader effort to enable rapid voxelization for complex models, which facilitates applications such as transparent illustration [S2, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models - GitHub Pages, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
[S2] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
