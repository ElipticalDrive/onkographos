# Voxel Volume Data Format

Also called: JSON Voxel Volume Format

## Overview
Voxel Volume Data Format is a JSON-based specification for storing structured voxel grid data, primarily designed for interoperability between voxelization tools and rendering engines. It solves the problem of defining volumetric primitives with associated attributes (such as material index or density) and spatial bounds in a human-readable, extensible format. It differs from binary list formats like VL32 by using a structured schema with semantic attribute definitions and normalized component types, whereas VL32 uses a flat binary struct array [S1, unofficial][S2].

## How it works
**Inputs:** A set of voxel primitives with spatial coordinates, dimensions, and associated scalar or vector attributes.
**Outputs:** A JSON object representing the volume data.

**Data Structures:**
The root object contains the following fields:
1.  `attributes`: An array of attribute definition objects. Each object defines a semantic property (e.g., "MATERIAL_INDEX", "DENSITY") and its data type.
    *   `semantic` (string): The name of the attribute.
    *   `componentType` (integer): The numeric type code (e.g., 5123 for `UNSIGNED_BYTE`).
    *   `normalized` (boolean, optional): Whether the component values are normalized to a standard range (e.g., 0.0–1.0 for floats, 0–1 for integers).
2.  `position` (array of 3 numbers): The origin coordinates $(x_0, y_0, z_0)$ of the entire volume.
3.  `size` (array of 3 numbers): The total dimensions $(W, H, D)$ of the volume in voxels.
4.  `primitives`: An array of primitive objects. Each primitive represents a sub-volume or chunk.
    *   `position` (array of 3 numbers): Local origin relative to the volume origin.
    *   `size` (array of 3 numbers): Local dimensions.
    *   `materialType` (string, optional): Identifier for the material palette.
    *   Data arrays for each attribute defined in `attributes`, stored in row-major order (x, then y, then z).

**Steps:**
1.  Define the global `attributes` array specifying the semantic and type for each voxel property.
2.  Set the global `position` and `size` to define the bounding box of the entire volume.
3.  For each primitive (chunk) within the volume:
    a. Define its local `position` and `size`.
    b. Populate the data arrays for each attribute. The data is stored as a flat array of values corresponding to the voxels in the primitive's bounding box.
    c. If `normalized` is true for an attribute, ensure values are scaled appropriately for the `componentType`.

**Formulas:**
For a primitive with local dimensions $W_p, H_p, D_p$, the number of voxels $N$ is:
$$ N = W_p \times H_p \times D_p $$
The index $k$ of a voxel at local coordinates $(x, y, z)$ within the primitive's data array is:
$$ k = x + y \cdot W_p + z \cdot W_p \cdot H_p $$

**Parameters:**
*   `componentType`: Uses numeric codes consistent with WebGL/Khronos standards (e.g., 5123 for `UNSIGNED_BYTE`) [S1, unofficial].
*   `normalized`: Boolean flag indicating value scaling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** When using `normalized` floats, ensure the `componentType` supports the required precision.
*   **Boundaries:** Primitive positions and sizes must be consistent with the global volume bounds.
*   **Empty Voxels:** Unlike VL32, this format does not explicitly define a "void" marker in the base schema; emptiness is typically handled by material index 0 or alpha channels if present [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_voxel_volume(attributes, global_position, global_size, primitives):
    """
    Construct a Voxel Volume Data Format JSON object.
    
    Args:
        attributes: List of dicts with 'semantic', 'componentType', and optional 'normalized'.
        global_position: [x, y, z] origin of the volume.
        global_size: [width, height, depth] of the volume.
        primitives: List of dicts with 'position', 'size', and attribute data arrays.
        
    Returns:
        dict: The JSON-serializable volume object.
    """
    volume = {
        "attributes": attributes,
        "position": global_position,
        "size": global_size,
        "primitives": []
    }
    
    for prim in primitives:
        primitive_obj = {
            "position": prim["position"],
            "size": prim["size"]
        }
        
        # Add attribute data arrays
        for attr in attributes:
            semantic = attr["semantic"]
            if semantic in prim:
                primitive_obj[semantic] = prim[semantic]
                
        volume["primitives"].append(primitive_obj)
        
    return volume

def get_voxel_index(local_x, local_y, local_z, width, height):
    """
    Calculate the linear index in a flat array for a 3D voxel grid.
    """
    return local_x + local_y * width + local_z * width * height
```

## Subtopics

### VL32 File Format
VL32 is a binary intermediate file format used for benchmarking compression efficiency [S2]. It consists of a single struct `main` containing an array of `voxel` structs, with no header [S2]. Each `voxel` struct contains four fields: `x` (i32), `y` (i32), `z` (i32), and `color` (argb32) [S2]. Coordinates are big-endian two's complement 32-bit integers [S2]. The `color` struct contains four unsigned 8-bit integers: `alpha`, `red`, `green`, `blue` [S2]. Voxels with an alpha value of zero are treated as void [S2]. It is used for compression benchmarking because it has no header and matches typical in-memory representations like `std::vector` [S2].

### Zoxel Format
Zoxel is a JSON-based format for storing voxel data, often used in web-based voxel engines [S3, unofficial]. The file structure is a JSON object with fields `creator` (string), `width`/`height`/`depth` (integers), `version` (integer, typically 1), `frames` (integer), and `frame1` (array of voxel entries) [S3, unofficial]. Each entry in `frame1` is an array `[x, y, z, id, meta]` [S3, unofficial]. IDs are stored multiplied by 100 to preserve fractional precision during JSON serialization [S3, unofficial]. Importing requires dividing the ID by 100 [S3, unofficial]. Exporting involves converting the ID to an RGB color using a coloring function, converting to hex, appending `FF` for alpha if length <= 6, and parsing as a 32-bit integer [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Parsing is linear in the number of voxels for JSON formats. Binary formats like VL32 are faster due to lack of parsing overhead.
*   **Memory Complexity:** Proportional to the number of voxels and attributes. JSON formats have higher overhead due to text representation.
*   **Strengths:** Human-readable, extensible via attributes, standard JSON parsing tools available.
*   **Limitations:** Large file sizes for dense volumes, slower parsing than binary formats.
*   **When to use:** Interchange between tools, debugging, small to medium volumes.
*   **When not to use:** High-performance real-time rendering of large volumes, compression benchmarking (use VL32).

## Background
The Voxel Volume Data Format (JSON Voxel Volume Format) was proposed as an unofficial specification for voxel data interchange [S1, unofficial]. VL32 was developed as part of a voxel compression research project to provide a simple binary baseline for compression algorithms [S2]. Zoxel format is used in the CEWBS (Cross-Engine Web-Based Sandbox) project for web-based voxel rendering [S3, unofficial].

## Sources
[S1] Voxel data format, https://github.com/enkisoftware/OpenVoxelFileFormat/issues/4, fetched 2026-10-27 (unofficial)
[S2] 32-Bit Voxel List (VL32), https://eisenwave.github.io/voxel-compression-docs/file_formats/vl32.html, fetched 2026-10-27
[S3] wwwredfish/CEWBS, https://github.com/wwwredfish/CEWBS, fetched 2026-10-26 (unofficial)
