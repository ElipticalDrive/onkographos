# Voxelizer Initialization

Also called: Voxelizer Construction, Voxel Model Creation

## Overview
Voxelizer Initialization creates a `Voxelizer` instance, either empty or with dimensions derived from a mesh's axis-aligned bounding box (AABB) [S1, unofficial]. It serves as the foundational step in producing volume data by establishing the spatial structure (`VoxModel`) and memory constraints before any geometry is processed [S1, unofficial]. It differs from subsequent voxelization steps (which fill the grid) by focusing solely on the allocation and configuration of the underlying data structures [S1, unofficial].

## How it works
**Inputs:** A source mesh (for dimension derivation), `max_depth`, `chunk_world_size`, and `memory_budget`.
**Outputs:** An initialized `Voxelizer` instance containing a `VoxModel`.

**Data Structures:**
*   `Voxelizer`: The top-level instance holding the voxel model and interning state.
*   `VoxModel`: The core data structure containing:
    *   `dimensions`: Width, height, and depth in voxels (derived from mesh AABB).
    *   `max_depth`: Integer specifying the maximum depth of the voxel tree.
    *   `chunk_world_size`: Float specifying the size of each chunk in world units.
    *   `memory_budget`: Float or integer specifying the allocated memory limit.

**Steps:**
1.  **Calculate World Bounds (if derived from mesh):**
    If initializing from a mesh, compute the world bounds dimensions. For each axis ($x, y, z$):
    $$ \text{world\_bounds}_{axis} = \lceil \text{mesh.size}_{axis} \rceil + 1 $$
    where $\lceil \cdot \rceil$ is the ceiling function and the result is cast to a 32-bit signed integer (`i32`) [S1, unofficial].

2.  **Create VoxModel:**
    Instantiate the `VoxModel` using the calculated dimensions (or default empty dimensions if no mesh is provided), along with the provided parameters: `max_depth`, `chunk_world_size`, and `memory_budget` [S1, unofficial].

3.  **Initialize Voxelizer:**
    Construct the `Voxelizer` instance, embedding the `VoxModel` and initializing any necessary internal state (e.g., interner for chunk management) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_voxelizer(mesh=None, max_depth: int = 8, chunk_world_size: float = 1.0, memory_budget: float = 1e9):
    """
    Creates a Voxelizer instance.
    
    Args:
        mesh: Optional mesh object with a .size attribute (vec3). If None, creates an empty voxelizer.
        max_depth: Maximum depth of the voxel tree.
        chunk_world_size: Size of each chunk in world units.
        memory_budget: Memory budget for the voxel model.
        
    Returns:
        Voxelizer instance.
    """
    if mesh is not None:
        # Calculate world bounds from mesh size
        # Cast to i32 after ceiling
        world_bounds_x = int(math.ceil(mesh.size.x)) + 1
        world_bounds_y = int(math.ceil(mesh.size.y)) + 1
        world_bounds_z = int(math.ceil(mesh.size.z)) + 1
        dimensions = (world_bounds_x, world_bounds_y, world_bounds_z)
    else:
        # Not given in the sources: Define default empty dimensions (e.g., 0 or 1)
        dimensions = (0, 0, 0)

    # Create VoxModel with calculated dimensions and parameters
    vox_model = VoxModel(
        dimensions=dimensions,
        max_depth=max_depth,
        chunk_world_size=chunk_world_size,
        memory_budget=memory_budget
    )

    # Initialize Voxelizer instance
    voxelizer = Voxelizer(vox_model=vox_model)
    
    # Not given in the sources: Initialize interner or other internal state if required
    # voxelizer.interner = Intern() 
    
    return voxelizer
```

**Parameters:**
*   `max_depth`: The maximum depth of the voxel tree [S1, unofficial]. Typical value: Not given in the sources.
*   `chunk_world_size`: The size of each chunk in world units [S1, unofficial]. Typical value: Not given in the sources.
*   `memory_budget`: The memory budget for the voxel model [S1, unofficial]. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation of world bounds involves casting floating-point mesh sizes to integers. Using `ceil()` ensures the bounds fully encompass the mesh, but the `+1` offset may introduce a 1-voxel padding depending on implementation specifics [S1, unofficial].
*   **Empty Input:** If no mesh is provided, the dimensions must be handled gracefully (e.g., zero-sized or default grid) [S1, unofficial].

## Subtopics

### T-SDF Integration and Carving Voxelization
A specialized initialization and refinement pipeline used in cage building for 3D Gaussian Splatting deformation [S2, unofficial]. It differs from standard AABB-based initialization by employing a fixed-resolution grid (128) and integrating T-SDF carving [S2, unofficial]. Choose this method when high-fidelity surface representation is required for deformation cages [S2, unofficial].
*   **Steps:** Create a voxel grid with resolution 128 [S2, unofficial]. Employ T-SDF integration and carving procedure from Open3D using default parameters [S2, unofficial]. Perform alternating decimation: Stage 1 collapses 10 edges, Stage 2 runs for 10 gradient-descent steps [S2, unofficial]. Run MVC refinement stage using Adam optimizer with learning rate 0.005 [S2, unofficial].
*   **Parameters:** Alternating decimation start: Begin alternating at 1000 vertices [S2, unofficial]. Surface sampling: Sample $|S| = 256$ points on the object surface [S2, unofficial]. Decimation parameters: Set $\mu = 100$, $\rho = 10^{-4}$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Determined by `memory_budget` and `chunk_world_size` [S1, unofficial].
*   **Strengths:** Provides a structured, memory-bounded foundation for voxel-based rendering [S1, unofficial].
*   **Limitations:** Dimension derivation from mesh AABB may result in sparse grids if the mesh is small relative to the world [S1, unofficial].
*   **When to use:** When setting up a voxelizer for mesh-based volume data [S1, unofficial].
*   **When not to use:** When using fixed-resolution grids for surface reconstruction (use T-SDF Integration instead) [S2, unofficial].

## Background
Voxelizer Initialization is a component of the `voxelis` library, an unofficial voxelization tool [S1, unofficial]. It is used in procedural generation and voxel-based rendering pipelines [S1, unofficial]. The T-SDF variant is used in GSDeformer for cage-based deformation of 3D Gaussian Splatting [S2, unofficial].

## Sources
[S1] WildPixelGames/voxelis, https://github.com/WildPixelGames/voxelis, fetched 2026-10-26 (unofficial)
[S2] GSDeformer: Direct Cage-based Deformation for 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2405.15491, fetched 2026-10-30 (unofficial)
