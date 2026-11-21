# Far Terrain LOD with Voxel Coverage Masking

Also called: Far Terrain

## Overview
Far Terrain LOD with Voxel Coverage Masking is a rendering technique for distant terrain that uses multiple levels of detail (LOD) meshes, discarding geometry where voxel chunks are present to prevent visual artifacts [S1, unofficial]. It employs a coverage mask to handle boundary transitions between the static far terrain and dynamic voxel chunks, specifically preventing z-fighting between LOD levels and sky leaks where voxels meet the terrain [S1, unofficial]. Unlike voxel chunks which are dynamically meshed per chunk, far terrain uses pre-generated static meshes with LOD, relying on hole rects and mask sampling to discard geometry in covered areas [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs**: Player position $(px, pz)$, voxel chunk state data (mesh presence and missing status), LOD level index $l$.
*   **Outputs**: Visible far terrain mesh geometry, updated coverage mask texture, hole rectangle for finer levels.

### Data Structures

**Level State**
Each LOD level $l$ maintains the following state:
*   `step`: Grid step size, calculated as $BASE\_STEP \ll l$.
*   `half`: Half-width of the mesh grid, $N \times step / 2$.
*   `snap`: Snap distance for centering, $step \times 8$.
*   `cx`, `cz`: Current center coordinates in world space.
*   `wx`, `wz`: Previous world center coordinates (NaN if not yet initialized).
*   `ready`: Boolean indicating if the mesh is loaded and ready.
*   `mesh`: Reference to the `THREE.Mesh` object.
*   `hole`: A `THREE.Vector4` representing the rectangle $(x_0, z_0, x_1, z_1)$ of the finer level's coverage area, used to define the hole for the current level.

**Coverage Mask**
*   A texture (or array) storing coverage status for columns in the scan window.
*   Values: 0 (uncovered/empty), 128 (edge/transition), 255 (covered).

### Parameters
*   `N`: Grid dimension (size of the mesh in vertices/indices). Not given in the sources.
*   `BASE_STEP`: Base step size for LOD 0. Not given in the sources.
*   `LEVELS`: Total number of LOD levels. Not given in the sources.
*   `MASK_R`: Radius of the scan window for updating the coverage mask. Not given in the sources.
*   `SKIRT`: Constant value $128.0$ used to sink vertices inside meshed voxel columns to prevent view rays from slipping between voxel chunks and far terrain [S1, unofficial].

### Algorithm Steps

1.  **Initialization**:
    For each level $l$ from $0$ to $LEVELS-1$:
    *   Calculate `step` = $BASE\_STEP \ll l$.
    *   Calculate `half` = $N \times step / 2$.
    *   Calculate `snap` = $step \times 8$.
    *   If $l > 0$, calculate `skip` = $\lfloor (f.half - 1.5 \times (f.snap + snap) - step) / step \rfloor$. [G: `f` refers to the current level's parameters].
    *   Generate an index buffer for an $N \times N$ grid. Alternate diagonals based on $(i + j) \& 1$ to prevent flat-shading artifacts.
    *   Create a `BufferGeometry` with position (Float32, 3), color (Uint8, 4), and index.
    *   Create a `hole` Vector4 initialized to $(1, 1, 0, 0)$.
    *   Create a `MeshLambertMaterial` with custom shader injection.
    *   Set mesh properties: `frustumCulled=false`, `visible=false`, `receiveShadow=true`, `castShadow=false`, `matrixAutoUpdate=false`, `renderOrder=-1 + l \times 0.01`.

2.  **Material Setup (for $l > 0$)**:
    *   Enable polygon offset to prevent depth ties with finer levels: `polygonOffset=true`, `polygonOffsetFactor=1+l`, `polygonOffsetUnits=2+2*l`.
    *   **Vertex Shader Injection**:
        *   Define a `farMask` function sampling the `uFarMask` texture.
        *   In `begin_vertex`, compute world position `wp`.
        *   Sample 4 corners of `farMask`.
        *   Take the minimum sample value. If `min > 0.25`, subtract `SKIRT` (128) from the y-coordinate.
    *   **Fragment Shader Injection**:
        *   Discard the fragment if the vertex world position is inside the hole rect: $uHole.x < vWPos.x < uHole.z$ AND $uHole.y < vWPos.z < uHole.w$.
        *   Discard the fragment if the `farMask` sample $> 0.75$ (edge column kept at 0.5, sunk).

3.  **Update Loop (per frame)**:
    *   Call `updateMask(px, pz)`.
    *   For each level $l$:
        *   If player distance from level center $(wx, wz) \le snap$ AND $wx$ is not NaN, skip this level.
        *   Else, calculate new center:
            *   $cx = \text{round}(px / snap) \times snap$
            *   $cz = \text{round}(pz / snap) \times snap$
        *   Update $wx, wz$ to $cx, cz$.
        *   Calculate grid bounds: $x_0 = cx - half$, $z_0 = cz - half$.
        *   Request a meshing job with type `'far'`, id $0$, $x_0$, $z_0$, $step$, $n=N$.
        *   Define a callback `install(l, cx, cz, result)`.

4.  **Install Mesh (Callback)**:
    *   Update position and color buffers from `result`.
    *   Set $cx, cz$, `ready=true`.
    *   Set mesh position to $(cx - half, -1.5, cz - half)$.
    *   Update matrix.
    *   Set `visible=true`.
    *   Update hole for next coarser level ($l+1$):
        *   $m = half - up.step$ (where `up` is the next level).
        *   Set `up.hole` to $(cx - m, cz - m, cx + m, cz + m)$.

5.  **UpdateMask(px, pz)**:
    *   Initialize `cov` array to 0.
    *   Scan columns in range $[pcx - MASK\_R, pcx + MASK\_R] \times [pcz - MASK\_R, pcz + MASK\_R]$.
    *   For each column $ci$:
        *   If `colState[ci] == 2`:
            *   Check chunks: count meshes and missing.
            *   Covered if `meshes > 0` AND `missing == 0`.
            *   If `meshes == 0`, remove chunks from settled set.
            *   Store covered status in `cov` array.
    *   **Mask Value Calculation**:
        *   For each column in scan window:
            *   If `cov[k]` is true, set $v=255$.
            *   Check 3x3 neighborhood in `cov` array; if any neighbor is uncovered (`cov[neighbor] == 0`), set $v=128$.
            *   Else $v=255$.
            *   If $d[ci] \neq v$, update $d[ci]$ and set `changed=true`.
    *   Clear texels outside scan window (set to 0).
    *   Update mask if `changed`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

# Constants
SKIRT = 128.0
MASK_R = 10  # Example value, not given in sources
N = 64       # Example value, not given in sources
BASE_STEP = 100  # Example value, not given in sources
LEVELS = 4       # Example value, not given in sources

class FarTerrainLevel:
    def __init__(self, l):
        self.l = l
        self.step = BASE_STEP << l
        self.half = N * self.step / 2
        self.snap = self.step * 8
        self.cx = 0.0
        self.cz = 0.0
        self.wx = float('nan')
        self.wz = float('nan')
        self.ready = False
        self.hole = np.array([1, 1, 0, 0])  # x0, z0, x1, z1
        self.mesh = None  # Placeholder for mesh object
        self.skip = 0
        if l > 0:
            # f refers to current level params in source, using self
            self.skip = math.floor((self.half - 1.5 * (self.snap + self.snap) - self.step) / self.step)

    def update_center(self, px, pz):
        if self.wx == self.wx:  # Check not NaN
            dist = math.sqrt((px - self.wx)**2 + (pz - self.wz)**2)
            if dist <= self.snap:
                return False  # Skip update
        
        self.cx = round(px / self.snap) * self.snap
        self.cz = round(pz / self.snap) * self.snap
        self.wx = self.cx
        self.wz = self.cz
        return True

    def get_grid_bounds(self):
        return self.cx - self.half, self.cz - self.half

def update_mask(px, pz, col_state, cov_array):
    pcx = round(px / MASK_R) * MASK_R
    pcz = round(pz / MASK_R) * MASK_R
    
    # Initialize cov array to 0
    cov = np.zeros((2 * MASK_R + 1, 2 * MASK_R + 1))
    
    # Scan window
    for i in range(-MASK_R, MASK_R + 1):
        for j in range(-MASK_R, MASK_R + 1):
            ci = (i + MASK_R, j + MASK_R) # Index into cov
            # Check colState at this column
            # Note: col_state is a global map, simplified here
            if col_state.get((pcx + i, pcz + j)) == 2:
                meshes = count_meshes(pcx + i, pcz + j)
                missing = count_missing(pcx + i, pcz + j)
                if meshes > 0 and missing == 0:
                    cov[ci] = 1
                elif meshes == 0:
                    remove_chunks(pcx + i, pcz + j)
    
    # Mask value calculation
    d = np.zeros_like(cov)
    changed = False
    for i in range(-MASK_R, MASK_R + 1):
        for j in range(-MASK_R, MASK_R + 1):
            ci = (i + MASK_R, j + MASK_R)
            v = 0
            if cov[ci] == 1:
                v = 255
                # Check 3x3 neighborhood
                for ni in range(-1, 2):
                    for nj in range(-1, 2):
                        ni_ci = (i + ni + MASK_R, j + nj + MASK_R)
                        if 0 <= ni_ci[0] < cov.shape[0] and 0 <= ni_ci[1] < cov.shape[1]:
                            if cov[ni_ci] == 0:
                                v = 128
                                break
                    if v == 128:
                        break
            else:
                v = 0 # Uncovered
            
            if d[ci] != v:
                d[ci] = v
                changed = True
    
    # Clear texels outside scan window
    # (Assuming d is only updated within window, outside is 0 by init)
    
    if changed:
        update_texture(d)

def install_mesh(level, cx, cz, result):
    level.mesh.update_buffers(result)
    level.cx = cx
    level.cz = cz
    level.ready = True
    level.mesh.position = (cx - level.half, -1.5, cz - level.half)
    level.mesh.update_matrix()
    level.mesh.visible = True
    
    # Update hole for next coarser level (l+1)
    # Note: This assumes access to the next level's object
    # In a real system, this would be handled by the parent or a registry
    pass

def vertex_shader_injection(wp, far_mask_texture, hole):
    # Sample 4 corners of farMask
    corners = sample_farmask_corners(wp, far_mask_texture)
    min_sample = min(corners)
    
    if min_sample > 0.25:
        wp.y -= SKIRT
        
    # Discard if inside hole
    if hole[0] < wp.x < hole[2] and hole[1] < wp.z < hole[3]:
        discard()
        
    # Discard if farMask sample > 0.75
    if min_sample > 0.75:
        discard()
        
    return wp
```

## Operating characteristics

*   **Time Complexity**: The update loop runs per frame. Mask update is $O(MASK\_R^2)$ per frame. Mesh generation is asynchronous and depends on chunk density.
*   **Memory Complexity**: $O(N^2)$ per LOD level for the mesh geometry. Mask texture size is $O(MASK\_R^2)$.
*   **Performance**: Pre-generated static meshes reduce per-frame computation for distant terrain. Dynamic voxel chunks are handled separately.
*   **Strengths**: Prevents z-fighting and sky leaks. Efficient for large distances.
*   **Limitations**: Static meshes cannot adapt to dynamic changes in the far terrain (unless regenerated). Requires careful tuning of `SKIRT` and mask thresholds.
*   **When to use**: Distant terrain rendering where voxel chunks are sparse or absent.
*   **When not to use**: Close-range terrain where voxel chunks are dense.

## Background

*   **Origin**: Developed for the "2.5d-experiment" project, a browser-based voxel engine using Rust WASM [S1, unofficial].
*   **History**: Part of the effort to render 12.8B destructible voxels in the browser [S1, unofficial].
*   **Usage**: Used in the `src/far-terrain.ts` module of the project [S1, unofficial].

## Sources

[S1] 12.8B destructible voxels rendered in the browser with Rust WASM custom engine, https://github.com/cktang88/2.5d-experiment, fetched 2026-10-26 (unofficial)
