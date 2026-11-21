# 3D Gaussian Splatting Model Preprocessing and Grid Partitioning

Also called: PLY Gaussian Loading with Fallbacks

## Overview
This method preprocesses a raw 3D Gaussian Splatting model stored in PLY format for use in a specific rendering engine (Unreal Engine). It parses the PLY file, converts coordinate systems and property encodings to match the engine's internal representation, and partitions the Gaussians into a spatial grid for efficient culling and rendering. It differs from generic loaders by applying specific axis swaps, exponential scale decoding, and sigmoid opacity activation required by the target engine's shader pipeline.

## How it works
**Inputs:**
- `FilePath`: Relative path to the PLY file within the project content directory.
- `ProjectContentDir`: Base directory path for the application's content.

**Outputs:**
- `GaussianData`: A structured collection of Gaussian properties (Position, Scale, Rotation, Color, Opacity, Harmonics) organized by spatial grid cells.
- `GridMetadata`: Bounding box (`BoundsMin`, `BoundsMax`) and grid resolution (`CellsPerEdge`).

**Data Structures:**
- `FLinearColor`: A 4-component vector (R, G, B, A) used to pack multiple float properties.
- `FQuat`: A 4-component quaternion (X, Y, Z, W) for rotation.
- `GridCell`: A container holding lists of Gaussian indices or properties for a specific 3D grid coordinate $(i, j, k)$.

**Steps:**

1.  **Path Construction and Validation:**
    Construct the absolute file path:
    $$ \text{AbsolutePath} = \text{ProjectContentDir} + \text{"Models/"} + \text{FilePath} $$
    Initialize a PLY reader (e.g., `miniply::PLYReader`) with the absolute path. Validate that the reader successfully opened the file. If invalid, abort.

2.  **Header Parsing and Element Identification:**
    Iterate through the PLY header elements using `reader.has_element()` and `reader.next_element()`. Identify the vertex element by checking if `reader.element_is(miniply::kPLYVertexElement)`.

3.  **Data Extraction:**
    Load the vertex data using `reader.load_element()`. Retrieve the number of vertices:
    $$ N = \text{reader.num\_rows()} $$
    For each property in the vertex element, allocate a float array of size $N$ and extract the data:
    ```python
    columnData = new float[N]
    reader.extract_properties(property_indexes, 1, PLYPropertyType::Float, columnData)
    vertexData[property_name] = columnData
    ```
    Validate that the following required properties exist in `vertexData`:
    - Position: `x`, `y`, `z`
    - Orientation: `rot_0`, `rot_1`, `rot_2`, `rot_3`
    - Scale: `scale_0`, `scale_1`, `scale_2`
    - Opacity: `opacity`
    - Zero Order Harmonics: `f_dc_0`, `f_dc_1`, `f_dc_2`
    - Higher Order Harmonics (Optional): `f_rest_0` to `f_rest_44`

4.  **Bounding Box Calculation:**
    Initialize `BoundsMin` and `BoundsMax` with the first vertex's position, converted to Unreal coordinates:
    $$ P_{unreal} = 100.0 \times (x, -z, -y) $$
    Iterate through all $N$ vertices to update `BoundsMin` and `BoundsMax` component-wise. Compute `BoxEdgeLengths`:
    $$ L = \text{BoundsMax} - \text{BoundsMin} $$

5.  **Grid Resolution Determination:**
    Determine `CellsPerEdge` based on $N$:
    - If $N \le 1,500,000$, `CellsPerEdge` = 1.
    - If $1,500,000 < N \le 3,500,000$, `CellsPerEdge` = 2.
    - If $N > 3,500,000$, `CellsPerEdge` = 4.

6.  **Directory Creation:**
    Create necessary directories for the model and emitter data using a helper function `CreateDirectory`.

7.  **Gaussian Property Processing and Grid Assignment:**
    Initialize an empty grid structure of size `CellsPerEdge` $\times$ `CellsPerEdge` $\times$ `CellsPerEdge`. Iterate through each vertex $v$ from $0$ to $N-1$:

    a. **Position Conversion:**
       Convert to Unreal coordinates and scale:
       $$ P_{final} = 100.0 \times (x_v, -z_v, -y_v) $$
       Pack into `FLinearColor`:
       $$ \text{PosColor} = (P_{final}.x, P_{final}.y, P_{final}.z, 0.0) $$

    b. **Scale Conversion:**
       Apply exponential activation and axis swap:
       $$ s'_0 = \exp(\text{scale\_0}_v), \quad s'_1 = \exp(\text{scale\_2}_v), \quad s'_2 = \exp(\text{scale\_1}_v) $$
       Scale by 100.0 and pack into `FLinearColor`:
       $$ \text{ScaleColor} = (100.0 \times s'_0, 100.0 \times s'_1, 100.0 \times s'_2, 0.0) $$
       *Note: The source swaps `scale_1` and `scale_2` during packing.*

    c. **Rotation Conversion:**
       Construct quaternion from PLY order `(rot_1, rot_2, rot_3, rot_0)`:
       $$ q = \text{Quat}(\text{rot\_1}_v, \text{rot\_2}_v, \text{rot\_3}_v, \text{rot\_0}_v) $$
       Normalize $q$. Apply axis swap and sign flip:
       $$ q_{final} = (q.x, -q.z, -q.y, q.w) $$
       Pack into `FLinearColor` (using XYZW components):
       $$ \text{RotColor} = (q_{final}.x, q_{final}.y, q_{final}.z, q_{final}.w) $$

    d. **Color and Opacity Conversion:**
       Extract Zero Order Harmonics as BaseColor:
       $$ C_{base} = (\text{f\_dc\_0}_v, \text{f\_dc\_1}_v, \text{f\_dc\_2}_v) $$
       Apply sigmoid to opacity:
       $$ \alpha_v = \frac{1}{1 + \exp(-\text{opacity}_v)} $$
       Clamp $\alpha_v$ to $[0, 1]$. Pack into `FLinearColor`:
       $$ \text{ColorAlphaColor} = (C_{base}.x, C_{base}.y, C_{base}.z, \alpha_v) $$

    e. **Higher Order Harmonics (L1):**
       If `f_rest` properties exist, iterate $y$ from 0 to 9 with step 3. For each $y$, extract `f_rest_y`, `f_rest_y+1`, `f_rest_y+2` and pack into an `FLinearColor`. Append these to `harmonicsL1TextureData`.

    f. **Grid Indexing:**
       Calculate the grid index for the current vertex's `PosColor`:
       $$ i = \lfloor \frac{P_{final}.x - \text{BoundsMin}.x}{L.x} \times \text{CellsPerEdge} \rfloor $$
       $$ j = \lfloor \frac{P_{final}.y - \text{BoundsMin}.y}{L.y} \times \text{CellsPerEdge} \rfloor $$
       $$ k = \lfloor \frac{P_{final}.z - \text{BoundsMin}.z}{L.z} \times \text{CellsPerEdge} \rfloor $$
       Clamp $i, j, k$ to $[0, \text{CellsPerEdge}-1]$. Store the processed properties in the grid cell $(i, j, k)$.

**Edge Cases and Pitfalls:**
- **Precision:** The use of `FLinearColor` (typically 32-bit floats per channel) for packed data may introduce quantization errors for properties like rotation or high-precision harmonics.
- **Axis Swaps:** The specific axis swaps (Y/Z swap for position, 1/2 swap for scale, X/Z/Y permutation for rotation) are critical. Incorrect mapping results in rotated or scaled splats.
- **Harmonics Storage:** Higher order harmonics are stored in a separate texture-like structure (`harmonicsL1TextureData`), not in the main grid cell properties.

## Operating characteristics
- **Time Complexity:** $O(N)$ for parsing, conversion, and grid assignment, where $N$ is the number of Gaussians.
- **Memory Complexity:** $O(N)$ for storing the Gaussian properties and the grid structure. The grid itself is $O(C^3)$ where $C$ is `CellsPerEdge`, but typically $C^3 \ll N$.
- **Performance:** The method is a one-time preprocessing step. The grid partitioning enables $O(1)$ spatial queries during rendering.
- **Limitations:** The grid resolution is fixed based on vertex count thresholds, which may not be optimal for all scene densities. The coordinate system conversion is hardcoded for Unreal Engine's right-handed coordinate system.

## Background
This method originates from the `unreal-splat` project [S1, unofficial], an unofficial implementation of 3D Gaussian Splatting for Unreal Engine. It adapts the standard PLY format (which uses a left-handed coordinate system and raw scale/opacity values) to Unreal's requirements (right-handed, exponential scale, sigmoid opacity). The grid partitioning strategy is a heuristic based on vertex count to balance memory usage and culling efficiency.

## Sources
[S1] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
