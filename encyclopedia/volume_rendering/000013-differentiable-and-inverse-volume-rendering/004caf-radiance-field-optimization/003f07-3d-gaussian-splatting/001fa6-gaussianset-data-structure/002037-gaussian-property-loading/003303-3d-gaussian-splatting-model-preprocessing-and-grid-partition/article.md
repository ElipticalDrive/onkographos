# 3D Gaussian Splatting Model Preprocessing and Grid Partitioning

Also called: PLY Gaussian Loading with Fallbacks

## Overview
This method preprocesses a raw 3D Gaussian Splatting model stored in PLY format for use in Unreal Engine [S1, unofficial]. It parses the PLY file, converts coordinate systems and property encodings to match the engine's internal representation, and partitions the Gaussians into a spatial grid for efficient culling and rendering [S1, unofficial]. It differs from generic loaders by applying specific axis swaps, exponential scale decoding, and sigmoid opacity activation required by the target engine's shader pipeline [S1, unofficial].

## How it works
**Inputs:**
- `FilePath`: Relative path to the PLY file within the project content directory [S1, unofficial].
- `ProjectContentDir`: Base directory path for the application's content [S1, unofficial].

**Outputs:**
- `GaussianData`: A structured collection of Gaussian properties (Position, Scale, Rotation, Color, Opacity, Harmonics) organized by spatial grid cells [S1, unofficial].
- `GridMetadata`: Bounding box (`BoundsMin`, `BoundsMax`) and grid resolution (`CellsPerEdge`) [S1, unofficial].

**Data Structures:**
- `FLinearColor`: A 4-component vector (R, G, B, A) used to pack multiple float properties [S1, unofficial].
- `FQuat`: A 4-component quaternion (X, Y, Z, W) for rotation [S1, unofficial].
- `GridCell`: A container holding lists of Gaussian properties for a specific 3D grid coordinate $(i, j, k)$ [S1, unofficial].

**Steps:**

1.  **Path Construction and Validation:**
    Construct the absolute file path:
    $$ \text{AbsolutePath} = \text{ProjectContentDir} + \text{"Models/"} + \text{FilePath} $$
    Initialize a PLY reader with the absolute path and validate that it successfully opened the file [S1, unofficial]. If invalid, abort [S1, unofficial].

2.  **Header Parsing and Element Identification:**
    Iterate through the PLY header elements using `reader.has_element()` and `reader.next_element()` [S1, unofficial]. Identify the vertex element by checking if `reader.element_is(miniply::kPLYVertexElement)` [S1, unofficial].

3.  **Data Extraction:**
    Load the vertex data using `reader.load_element()` and retrieve the number of vertices $N = \text{reader.num\_rows()}$ [S1, unofficial]. For each property in the vertex element, allocate a float array of size $N$ and extract the data into a map keyed by property name [S1, unofficial]. Validate that the following required properties exist in the map: Position (`x`, `y`, `z`), Orientation (`rot_0`..`rot_3`), Scale (`scale_0`..`scale_2`), Opacity, Zero Order Harmonics (`f_dc_0`..`f_dc_2`), and optionally Higher Order Harmonics (`f_rest_0`..`f_rest_44`) [S1, unofficial].

4.  **Bounding Box Calculation:**
    Initialize `BoundsMin` and `BoundsMax` with the first vertex's position converted to Unreal coordinates:
    $$ P_{\text{unreal}} = 100.0 \times (x, -z, -y) $$
    Iterate through all $N$ vertices to update `BoundsMin` and `BoundsMax` component-wise [S1, unofficial]. Compute `BoxEdgeLengths`:
    $$ L = \text{BoundsMax} - \text{BoundsMin} $$
    All values are scaled by $100.0$ [S1, unofficial].

5.  **Grid Resolution Determination:**
    Determine `CellsPerEdge` based on $N$:
    - If $N \le 1,500,000$, `CellsPerEdge` = 1 [S1, unofficial].
    - If $1,500,000 < N \le 3,500,000$, `CellsPerEdge` = 2 [S1, unofficial].
    - If $N > 3,500,000$, `CellsPerEdge` = 4 [S1, unofficial].

6.  **Directory Creation:**
    Create necessary directories for the model and emitter data using a helper function `CreateDirectory` [S1, unofficial].

7.  **Gaussian Property Processing and Grid Assignment:**
    Initialize an empty grid structure of size `CellsPerEdge` $\times$ `CellsPerEdge` $\times$ `CellsPerEdge` [S1, unofficial]. Iterate through each vertex $v$ from $0$ to $N-1$:
    a. **Position Conversion:** Convert to Unreal coordinates and scale by $100.0$. Pack into `FLinearColor` as $(R=x, G=-z, B=-y, A=0)$ [S1, unofficial].
    b. **Scale Conversion:** Apply exponential activation and axis swap: $s'_0 = \exp(\text{scale\_0}_v)$, $s'_1 = \exp(\text{scale\_2}_v)$, $s'_2 = \exp(\text{scale\_1}_v)$. Scale by $100.0$ and pack into `FLinearColor` as $(R=s'_0, G=s'_1, B=s'_2, A=0)$ [S1, unofficial].
    c. **Rotation Conversion:** Construct quaternion from PLY order `(rot_1, rot_2, rot_3, rot_0)`, normalize it, and apply axis swap/sign flip: $(X=q.x, Y=-q.z, Z=-q.y, W=q.w)$. Pack into `FLinearColor` [S1, unofficial].
    d. **Color and Opacity Conversion:** Extract Zero Order Harmonics as BaseColor. Apply sigmoid to opacity: $\alpha_v = \frac{1}{1 + \exp(-\text{opacity}_v)}$. Clamp $\alpha_v$ to $[0, 1]$. Pack into `FLinearColor` as $(R=C_{base}.x, G=C_{base}.y, B=C_{base}.z, A=\alpha_v)$ [S1, unofficial].
    e. **Higher Order Harmonics (L1):** If `f_rest` properties exist, iterate $y$ from $0$ to $9$ with step $3$. Extract `f_rest_y`, `f_rest_y+1`, `f_rest_y+2` and pack into an `FLinearColor`. Append to `harmonicsL1TextureData` [S1, unofficial].
    f. **Grid Indexing:** Calculate the grid index for the current vertex's position:
    $$ i = \left\lfloor \frac{P_{\text{final}}.x - \text{BoundsMin}.x}{L.x} \times \text{CellsPerEdge} \right\rfloor $$
    $$ j = \left\lfloor \frac{P_{\text{final}}.y - \text{BoundsMin}.y}{L.y} \times \text{CellsPerEdge} \right\rfloor $$
    $$ k = \left\lfloor \frac{P_{\text{final}}.z - \text{BoundsMin}.z}{L.z} \times \text{CellsPerEdge} \right\rfloor $$
    Clamp $i, j, k$ to $[0, \text{CellsPerEdge}-1]$. Store the processed properties in the grid cell $(i, j, k)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def preprocess_gaussians(file_path, project_content_dir):
    # 1. Path & Reader
    absolute_path = project_content_dir + "Models/" + file_path
    reader = miniply.PLYReader(absolute_path)  # [S1, unofficial]
    if not reader.valid():
        raise ValueError("PLY file invalid or missing")

    # 2. Header & Vertex
    num_vertices = 0
    vertex_data = {}
    for element in reader.elements():
        if element.is_vertex():
            element.load()
            num_vertices = reader.num_rows()
            for prop_name in element.properties():
                prop_idx = element.find_property(prop_name)
                column_data = [0.0] * num_vertices
                reader.extract_properties([prop_idx], 1, miniply.PLYPropertyType.FLOAT, column_data)
                vertex_data[prop_name] = column_data

    # 3. Validate Required Properties
    required = ["x", "y", "z", "rot_0", "rot_1", "rot_2", "rot_3",
                "scale_0", "scale_1", "scale_2", "opacity",
                "f_dc_0", "f_dc_1", "f_dc_2"]
    for req in required:
        if req not in vertex_data:
            raise ValueError(f"Missing required property: {req}")
    has_rest = "f_rest_0" in vertex_data

    # 4. Bounding Box (Unreal coords: 100.0 * (x, -z, -y))
    bounds_min = [0.0, 0.0, 0.0]
    bounds_max = [0.0, 0.0, 0.0]
    for v in range(num_vertices):
        px = 100.0 * vertex_data["x"][v]
        py = 100.0 * -vertex_data["z"][v]
        pz = 100.0 * -vertex_data["y"][v]
        if v == 0:
            bounds_min = [px, py, pz]
            bounds_max = [px, py, pz]
        else:
            bounds_min = [min(bounds_min[i], [px, py, pz][i]) for i in range(3)]
            bounds_max = [max(bounds_max[i], [px, py, pz][i]) for i in range(3)]
    box_edge_lengths = [bounds_max[i] - bounds_min[i] for i in range(3)]

    # 5. Grid Resolution
    if num_vertices <= 1500000:
        cells_per_edge = 1
    elif num_vertices <= 3500000:
        cells_per_edge = 2
    else:
        cells_per_edge = 4

    # 6. Directories
    create_directory_structure()  # [S1, unofficial]

    # 7. Process & Grid Assign
    grid = [[[] for _ in range(cells_per_edge)] for _ in range(cells_per_edge)]
    harmonics_l1 = []

    for v in range(num_vertices):
        # Position
        pos = [100.0 * vertex_data["x"][v], 100.0 * -vertex_data["z"][v], 100.0 * -vertex_data["y"][v]]
        pos_color = [pos[0], pos[1], pos[2], 0.0]

        # Scale
        s0 = math.exp(vertex_data["scale_0"][v])
        s1 = math.exp(vertex_data["scale_2"][v])
        s2 = math.exp(vertex_data["scale_1"][v])
        scale_color = [100.0 * s0, 100.0 * s1, 100.0 * s2, 0.0]

        # Rotation
        quat = normalize_quat([vertex_data["rot_1"][v], vertex_data["rot_2"][v],
                               vertex_data["rot_3"][v], vertex_data["rot_0"][v]])
        rot_color = [quat[0], -quat[2], -quat[1], quat[3]]

        # Color & Opacity
        base_color = [vertex_data["f_dc_0"][v], vertex_data["f_dc_1"][v], vertex_data["f_dc_2"][v]]
        opacity = 1.0 / (1.0 + math.exp(-vertex_data["opacity"][v]))
        opacity = max(0.0, min(1.0, opacity))
        color_alpha_color = [base_color[0], base_color[1], base_color[2], opacity]

        # Higher Order Harmonics
        if has_rest:
            for y in range(0, 10, 3):
                h = [vertex_data[f"f_rest_{y}"][v], vertex_data[f"f_rest_{y+1}"][v], vertex_data[f"f_rest_{y+2}"][v], 0.0]
                harmonics_l1.append(h)

        # Grid Index
        idx = get_grid_index(pos, bounds_min, box_edge_lengths, cells_per_edge)
        grid[idx[0]][idx[1]].append({
            "pos": pos_color, "scale": scale_color, "rot": rot_color,
            "color": color_alpha_color
        })

    return grid, harmonics_l1

def get_grid_index(pos, bounds_min, edge_lengths, cells_per_edge):
    i = int(math.floor((pos[0] - bounds_min[0]) / edge_lengths[0] * cells_per_edge))
    j = int(math.floor((pos[1] - bounds_min[1]) / edge_lengths[1] * cells_per_edge))
    k = int(math.floor((pos[2] - bounds_min[2]) / edge_lengths[2] * cells_per_edge))
    return [max(0, min(cells_per_edge - 1, idx)) for idx in [i, j, k]]

def normalize_quat(q):
    norm = math.sqrt(sum(c*c for c in q))
    return [c/norm for c in q]
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `FLinearColor` (typically 32-bit floats per channel) for packed data may introduce quantization errors for properties like rotation or high-precision harmonics [S1, unofficial].
- **Axis Swaps:** The specific axis swaps (Y/Z swap for position, 1/2 swap for scale, X/Z/Y permutation for rotation) are critical [S1, unofficial]. Incorrect mapping results in rotated or scaled splats [S1, unofficial].
- **Harmonics Storage:** Higher order harmonics are stored in a separate texture-like structure (`harmonicsL1TextureData`), not in the main grid cell properties [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for parsing, conversion, and grid assignment, where $N$ is the number of Gaussians [G].
- **Memory Complexity:** $O(N)$ for storing the Gaussian properties and the grid structure [G]. The grid itself is $O(C^3)$ where $C$ is `CellsPerEdge`, but typically $C^3 \ll N$ [G].
- **Performance:** The method is a one-time preprocessing step [S1, unofficial]. The grid partitioning enables $O(1)$ spatial queries during rendering [G].
- **Limitations:** The grid resolution is fixed based on vertex count thresholds, which may not be optimal for all scene densities [S1, unofficial]. The coordinate system conversion is hardcoded for Unreal Engine's right-handed coordinate system [S1, unofficial].

## Background
This method originates from the `unreal-splat` project [S1, unofficial], an unofficial implementation of 3D Gaussian Splatting for Unreal Engine [S1, unofficial]. It adapts the standard PLY format (which uses a left-handed coordinate system and raw scale/opacity values) to Unreal's requirements (right-handed, exponential scale, sigmoid opacity) [S1, unofficial]. The grid partitioning strategy is a heuristic based on vertex count to balance memory usage and culling efficiency [S1, unofficial].

## Sources
[S1] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
