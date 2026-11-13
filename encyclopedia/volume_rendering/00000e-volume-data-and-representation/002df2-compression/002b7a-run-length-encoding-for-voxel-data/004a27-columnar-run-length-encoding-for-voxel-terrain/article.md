# Columnar Run-Length Encoding for Voxel Terrain

Also called: Columnar RLE, Columnar RLE for Voxel Terrain, Run-Length Encoding (RLE) for Voxel Terrain, VOXLAP RLE Voxel Storage, VxlCompiledRLE, VxlMap

## Overview
Columnar Run-Length Encoding (RLE) is a compression technique for voxel terrain where voxels are stored as vertical columns, compressing homogeneous runs of voxels along the y-axis (vertical) into single entries [S1, S2, unofficial]. It solves the problem of excessive memory usage and geometry overhead by storing only the compressed column data on the CPU and uploading individual mesh runs to the GPU, avoiding the need to store a byte or vertex for every individual voxel [S1, unofficial]. Unlike full 3D spatial subdivision methods (e.g., KD-trees or octrees) that compress in all three axes, this method performs 1D compression along the y-axis only, treating the problem as a 2D planar issue [S1, unofficial].

## How it works
**Inputs:** A voxel grid defined by dimensions $VSID \times VSID \times MAXZDIM$ (where $VSID$ is the x/y size and $MAXZDIM$ is the height).
**Outputs:** A compressed in-memory representation (VxlArena) and a set of GPU-uploaded meshes/uniforms for rendering.

**Data Structures:**
1.  **VxlColor:** A packed 4-byte struct `#[repr(C, packed)]` with fields `b: u8`, `g: u8`, `r: u8`, `shade: u8`. The `shade` field acts as a brightness multiplier: `0x00` $\rightarrow$ 0.0, `0x80` $\rightarrow$ 1.0, `0xFF` $\rightarrow$ 2.0 [S2, unofficial].
2.  **VxlState:** An enum with variants `Air`, `Unexposed`, and `Color(VxlColor)` [S2, unofficial].
3.  **VxlSlab:** A struct representing an inclusive z-range $[start, end]$, internally stored as a half-open interval $[start, end + 1)$. Fields: `start: u8`, `end: u8` [S2, unofficial].
4.  **VxlArena:** A memory arena for column storage.
    *   `VXL_ARENA_WORDS_PER_COLUMN`: 16 (32-bit words per column) [S2, unofficial].
    *   `VXL_ARENA_WORDS`: $VSID \times VSID \times 16$ (Total capacity, e.g., 4,194,304 words for $VSID=512$) [S2, unofficial].
    *   Uses a bit array (`VxlArenaBits`) to track used words and a data array (`VxlData`) for storage [S2, unofficial].
5.  **VxlArenaSpanRangePacked:** A 32-bit packed struct containing `start` (22 bits) and `len` (10 bits) to represent a range of words in the arena [S2, unofficial].

**Parameters:**
*   $VSID = 512$: Maximum dimensions in x and y [S2, unofficial].
*   $MAXZDIM = 64$: Maximum dimension in z (height) [S2, unofficial].
*   $MAX\_VERTICAL\_STACK = MAXZDIM / 2 = 32$ [S2, unofficial].

**Algorithm Steps:**
1.  **Column Initialization:** For each column $(x, z)$, initialize a `VxlSlab` context with `start = z_start` and `end = z_end + 1` [S2, unofficial].
2.  **Iteration:**
    *   Outer loop iterates $y$ from $y_{start}$ to $y_{end}$ inclusive [S2, unofficial].
    *   Inner loop iterates $x$ from $x_{start}$ to $x_{end}$ (Note: Source code example incorrectly uses $y_{end}$ in the loop bound, but logically iterates x) [S2, unofficial].
3.  **Edit Application:**
    *   Call `ctx.edit_column(x)` to get a mutable reference to the column [S2, unofficial].
    *   Determine if the operation is a solid block:
        *   If `solid` is true, call `column.insert(slab)` [S2, unofficial].
        *   If `solid` is false, call `column.remove(slab)` [S2, unofficial].
    *   After each $y$ iteration, call `ctx.advance_and_flush()` [S2, unofficial].
4.  **Finalization:** After loops complete, call `ctx.finish()` [S2, unofficial].
5.  **Mesh Generation (CPU Side):**
    *   Evaluate the RLE columns to fragment them into individual homogeneous runs [S1, unofficial].
    *   A run is defined by a sequence such as `6E1G9A` (6 earth, 1 grass, 9 air) from top to bottom [S1, unofficial].
    *   For each run, create a mesh with 8 vertices [S1, unofficial].
    *   Set up uniform variables for additional data required for that specific run [S1, unofficial].
6.  **GPU Upload:** Upload the individual runs and their uniform variables to the GPU [S1, unofficial].
7.  **CPU State Maintenance:** Keep the full RLE columns ("from fundament to heaven") on the CPU side to handle breaks in columns (e.g., caves or air pockets) where meshes must end and restart at different y-levels [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
VSID = 512
MAXZDIM = 64
VXL_ARENA_WORDS_PER_COLUMN = 16
VXL_ARENA_WORDS = VSID * VSID * VXL_ARENA_WORDS_PER_COLUMN

class VxlSlab:
    def __init__(self, start: int, end: int):
        # Internally stored as half-open [start, end + 1)
        self.start = start
        self.end = end + 1

class VxlColor:
    def __init__(self, b: int, g: int, r: int, shade: int):
        self.b = b
        self.g = g
        self.r = r
        self.shade = shade

class VxlArena:
    def __init__(self):
        self.data = [None] * VXL_ARENA_WORDS
        self.used = [False] * VXL_ARENA_WORDS

    def alloc(self, span_data: list) -> tuple:
        # Find an open spot in the arena, copy span_data, return (start, length)
        # Implementation details omitted as per source constraints
        pass

def set_rect(ctx, x_start, x_end, y_start, y_end, z_start, z_end, solid, slab_data):
    """
    Initializes a slab cache context and applies edits to columns.
    """
    slab = VxlSlab(start=z_start, end=z_end)
    
    # Outer loop iterates y
    for y in range(y_start, y_end + 1):
        # Inner loop iterates x (source code note: uses y_end incorrectly, corrected here)
        for x in range(x_start, x_end + 1):
            column = ctx.edit_column(x)
            if solid:
                column.insert(slab)
            else:
                column.remove(slab)
        ctx.advance_and_flush()
    ctx.finish()

def generate_meshes(columns):
    """
    Evaluates RLE columns and generates GPU-ready meshes.
    """
    meshes = []
    uniforms = []
    
    for x, z in columns:
        # Fragment column into individual runs
        runs = fragment_column(columns[x][z])
        
        for run in runs:
            # Create mesh with 8 vertices per run
            mesh = create_mesh_8_vertices(run)
            # Set up uniform variables for the run
            uniform = setup_uniforms(run)
            
            meshes.append(mesh)
            uniforms.append(uniform)
            
    return meshes, uniforms
```

## Subtopics

### [GPU Renderer for RL-compressed data](10812-gpu-renderer-for-rl-compressed-data.md)
A GPU-based volume rendering technique designed for data compressed via RLE, specifically targeting voxelized models with sharp or small details. It distinguishes between uncompressed surface voxels and compressed interior/exterior regions on-the-fly during ray casting, avoiding the overhead of reconstructing the full volume in memory.

## Operating characteristics
*   **Compression:** Compresses only in the y-axis (1D compression). Achieves compression ratios up to 1:20 for noise-free data [S1, unofficial].
*   **Performance Bias:** By biasing detail axially, maintaining steady framerates when viewing in different directions can become problematic [S1, unofficial].
*   **Fragmentation:** Storing lighting, cellular automata, or other non-material data in the same RLE container causes fragmentation because the data is no longer homogeneous, increasing the ratio of breaks [S1, unofficial]. It is recommended to upload separate runs for elements like lighting [S1, unofficial].
*   **Memory:** Uses a fixed-size arena. For $VSID=512$, the arena holds 4,194,304 words (16 MB) [S2, unofficial].
*   **Complexity:** The problem is reduced to a 2D planar problem, making it easier to consider than full 3D subdivision, but less efficient in spatial compression than KD-trees or octrees [S1, unofficial].

## Background
Columnar RLE is a reimplementation of VOXLAP's voxel storage format (.vxl map files) for CPU rendering [S2, unofficial]. VOXLAP was created by Ken Silverman [S2, unofficial]. The method was discussed in the context of rendering different materials in voxel terrain on GameDev Stack Exchange [S1, unofficial].

## Sources
[S1] Rendering different materials in a voxel terrain, https://gamedev.stackexchange.com/questions/45022/rendering-different-materials-in-a-voxel-terrain, fetched 2026-10-26 (unofficial)
[S2] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
