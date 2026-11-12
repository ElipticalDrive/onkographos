# Froxel Light Assignment

Also called: Froxelization, Light-to-Froxel Mapping

## Overview
Froxel Light Assignment is the process of mapping visible volumetric lights to a pre-constructed 3D froxel grid, producing a compact list of lights for each froxel. It solves the problem of efficiently determining which local lights (point and spot) contribute to each region of the camera frustum, enabling fast volumetric light scattering by skipping the sun and non-volumetric lights. It differs from [Froxel Grid Construction](13185-froxel-grid-construction.md) by operating on the light data rather than the spatial grid, and from [Froxel Light List Buffer Creation](13196-froxel-light-list-buffer-creation.md) by focusing on the assignment logic and offset computation rather than the final buffer layout.

## How it works
**Inputs:**
- A list of visible lights ($N_L$), filtered by frustum intersection. Each light has position and radius.
- A froxel grid defined by dimensions $G_X, G_Y, G_Z$ and view parameters (view/projection matrices, near/far planes).
- Maximum lights per froxel $M$.

**Outputs:**
- `FroxelLightCounts`: A structured buffer of size $G_X \times G_Y \times G_Z$ storing the number of lights assigned to each froxel.
- `FroxelLightOffsets`: A structured buffer of size $G_X \times G_Y \times G_Z$ storing the starting index in the global light index list for each froxel.
- `FroxelLightIndices`: A structured buffer of size $G_X \times G_Y \times G_Z \times M$ (or dynamically allocated based on total count) storing the light indices.

**Data Structures:**
- `FFroxelUniformParameters`: Contains `GridSize` (uint32: $G_X, G_Y, G_Z, G_X G_Y G_Z$), `ViewportSize`, `ViewRectMinPx`, `TileSizePx`, `InvTilePx`, `ViewMatrix`, `ProjectionMatrix`, `NearCm`, `FarCm`, `NumLights`, `MaxLightsPerFroxel`.
- `FFroxelLightData`: Light data structure (position, radius, etc.).
- `FroxelLists`: Contains `Offsets` (uint, size $G_X G_Y G_Z$), `Counts` (uint, size $G_X G_Y G_Z$), `TotalIndices` (uint), `LightIndices` (uint, size $G_X G_Y G_Z \times M$), `FroxelCount` (uint32).

**Steps:**

1. **Light Filtering:**
   For each light in the scene, check if its bounding sphere intersects the camera frustum. Only lights passing this test are included in the assignment process. Let $N_{vis}$ be the number of visible lights.

2. **Light Counting:**
   Execute a compute shader pass (`FFroxelCountCS`) to count the number of visible lights affecting each froxel.
   - Initialize `FroxelLightCounts` to zero.
   - For each visible light, determine which froxels it intersects.
   - Use atomic operations to increment the count for each intersected froxel in `FroxelLightCounts`.
   - Group count: $\lceil N_{vis} / 64 \rceil$.

3. **Offset Computation (Blelloch Scan):**
   Compute the starting index in the global light index list for each froxel using a 3-pass Blelloch scan approach. This involves computing offsets within Z-slices and then across Z-slices.
   - **Pass A:** Clear `ZSliceTotals` (size $G_Z$) and `FroxelLightOffsets` (size $G_X G_Y G_Z$) to zero. Execute `FFroxelOffsetPassACS` with group count $\lceil (G_X G_Y) / 256 \rceil$. This pass computes prefix sums within each Z-slice.
   - **Pass B:** Clear `ZSliceOffsets` (size $G_Z$) to zero. Execute `FFroxelOffsetPassBCS` with group count $\lceil G_Z / 1024 \rceil$. This pass computes prefix sums across Z-slices.
   - **Pass C:** Execute `FFroxelOffsetPassCCS` with group count $\lceil (G_X G_Y) / 1024 \rceil$. This pass adds the Z-slice group sums to each froxel's offset to get absolute indices.

4. **Light Assignment:**
   Execute a compute shader pass to write light indices into the `FroxelLightIndices` buffer.
   - For each visible light, determine which froxels it intersects.
   - Use atomic operations to write the light index into the `FroxelLightIndices` buffer at the position indicated by `FroxelLightOffsets` for the target froxel, then increment the offset.
   - Limit the number of lights per froxel to $M$. If a froxel exceeds $M$, additional lights are ignored or handled via overflow (not specified in sources).

5. **Debug Overlay (Optional):**
   Render a heatmap overlay showing froxel occupancy and light counts for debugging. This is done in a post-processing pass if `r.Froxel.Visualize` is enabled.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Light Filtering
visible_lights = []
for light in scene_lights:
    if frustum_intersects_sphere(light.position, light.radius):
        visible_lights.append(light)

num_visible = len(visible_lights)

# Step 2: Light Counting
froxel_counts = [0] * (GX * GY * GZ)

# Compute Shader: FFroxelCountCS
# Group Count: ceil(num_visible / 64)
for i in range(num_visible):
    light = visible_lights[i]
    # Determine intersected froxels for this light
    intersected_froxels = get_intersected_froxels(light.position, light.radius, GX, GY, GZ, view_params)
    for froxel_idx in intersected_froxels:
        atomic_add(froxel_counts[froxel_idx], 1)

# Step 3: Offset Computation
# Pass A
z_slice_totals = [0] * GZ
froxel_offsets = [0] * (GX * GY * GZ)

# Compute Shader: FFroxelOffsetPassACS
# Group Count: ceil((GX * GY) / 256)
for z in range(GZ):
    # Process each Z-slice
    slice_size = GX * GY
    # Blelloch scan within Z-slice
    # ... (implementation details of Blelloch scan)
    # Store results in froxel_offsets for the current Z-slice
    # Store slice totals in z_slice_totals

# Pass B
z_slice_offsets = [0] * GZ

# Compute Shader: FFroxelOffsetPassBCS
# Group Count: ceil(GZ / 1024)
# Blelloch scan across Z-slices
# ... (implementation details of Blelloch scan)
# Store results in z_slice_offsets

# Pass C
# Compute Shader: FFroxelOffsetPassCCS
# Group Count: ceil((GX * GY) / 1024)
for z in range(GZ):
    for y in range(GY):
        for x in range(GX):
            froxel_idx = z * (GX * GY) + y * GX + x
            # Add Z-slice offset to froxel offset
            froxel_offsets[froxel_idx] += z_slice_offsets[z]

# Step 4: Light Assignment
froxel_light_indices = [0] * (GX * GY * GZ * M)
current_indices = [0] * (GX * GY * GZ) # To track current write position

# Compute Shader: Light Assignment
for i in range(num_visible):
    light = visible_lights[i]
    intersected_froxels = get_intersected_froxels(light.position, light.radius, GX, GY, GZ, view_params)
    for froxel_idx in intersected_froxels:
        count = froxel_counts[froxel_idx]
        if count <= M:
            # Atomic write to froxel_light_indices
            idx = atomic_add(current_indices[froxel_idx], 1)
            if idx < M:
                froxel_light_indices[froxel_idx * M + idx] = i

# Step 5: Debug Overlay (Optional)
if visualize:
    render_heatmap(froxel_counts, froxel_offsets, froxel_light_indices)
```

**Parameters:**
- `r.Froxel.GridX`: Froxel grid dimension X. Default: 16.
- `r.Froxel.GridY`: Froxel grid dimension Y. Default: 16.
- `r.Froxel.GridZ`: Froxel grid dimension Z. Default: 16.
- `r.Froxel.MaxLightsPerFroxel`: Maximum number of lights that can be assigned to a single froxel. Default: 32.
- `r.Froxel.FarClipCm`: Froxel grid far clipping plane in cm. Default: 100000.
- `r.Froxel.ZMode`: Froxel grid Z distribution mode. 0: linear, 1: logarithmic. Default: 1.
- `r.Froxel.Visualize`: Visualize Froxel grid. 0: off, 1: heatmap, 2: hash pattern. Default: 0.
- `r.Froxel.DebugOpacity`: Overlay opacity when compositing on SceneColor. Default: 0.5f.

**Edge Cases and Pitfalls:**
- **Light Overflow:** If a froxel intersects more than $M$ lights, additional lights are ignored. This can lead to inaccurate lighting in dense light clusters.
- **Precision:** Floating-point precision issues in frustum intersection tests may cause lights to be incorrectly included or excluded.
- **Boundary Conditions:** Lights exactly on the frustum boundary may be inconsistently handled.
- **Atomic Contention:** High light density can lead to atomic contention in the counting and assignment passes, potentially reducing performance.

## Subtopics

### Froxel Offset Computation
Computes the offsets into the global light index list for each froxel using a 3-pass Blelloch scan approach. Results are stored in `FroxelLightOffsetsUAV`. It differs from the general assignment by focusing solely on the prefix sum computation. Choose it when you need to compute offsets for froxel light lists. [S1, unofficial]

### Froxel Light Counting
A compute shader pass that counts the number of lights affecting each froxel in the grid. The result is stored in the `FroxelLightCountsUAV` buffer. It differs from the general assignment by focusing solely on the counting step. Choose it when you need to determine the number of lights per froxel before assigning indices. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N_L \cdot K + G_X G_Y G_Z)$, where $N_L$ is the number of visible lights, $K$ is the average number of froxels intersected by a light, and $G_X G_Y G_Z$ is the number of froxels. The Blelloch scan is $O(G_X G_Y G_Z)$.
- **Memory Complexity:** $O(G_X G_Y G_Z \cdot M)$ for the light indices buffer, plus $O(G_X G_Y G_Z)$ for counts and offsets.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of visible lights and froxel grid resolution. Higher grid resolution increases memory usage and computation time.
- **Strengths:** Efficiently maps lights to spatial regions, enabling fast volumetric lighting.
- **Limitations:** Limited by the maximum number of lights per froxel. Atomic contention can reduce performance in dense scenes.
- **When to Use:** Real-time volumetric lighting with local lights.
- **When Not to Use:** Scenes with very high light density or where precise light contribution per froxel is critical and $M$ is insufficient.

## Background
Froxel Light Assignment is part of the Froxel Lighting system, an unofficial implementation for Unreal Engine 5. It was developed by Armin Tajik and published on GitHub. The technique is designed to optimize volumetric light scattering by clustering lights into froxels. [S1, unofficial]

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
