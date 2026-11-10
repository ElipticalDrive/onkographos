# Voxel File I/O

Also called: Output Format Handling, Voxel Export

## Overview
Voxel File I/O is the method for serializing and deserializing 3D voxel grids to and from persistent storage formats. It solves the problem of transferring volumetric data between generation pipelines (such as voxelization or neural radiance field training) and rendering engines by defining standard layouts for dimensions, metadata, and voxel values. Unlike the specific binary RLE compression of [Binvox File Parsing](2862-binvox-file-parsing.md) or the mesh-based export of [PLY File Parsing](2781-ply-file-parsing.md), this method encompasses a variety of raw and structured binary/textual formats, prioritizing direct access to voxel arrays or compatibility with specific visualization tools.

## How it works
The method varies significantly by target format. The following subsections detail the specific implementations found in the source literature.

### CSV Export
Exports voxel data as a comma-separated values text file.
- **Inputs**: A list of voxel coordinates $(x, y, z)$ and an associated scalar value (e.g., `phase`).
- **Outputs**: A `.csv` file.
- **Steps**:
    1. Write the header row: `x,y,z,phase`.
    2. Iterate through each voxel.
    3. Write a row containing the integer coordinates and the scalar value.
- **Pitfall**: This format is extremely verbose and inefficient for large volumes due to text overhead and lack of spatial locality.

### PLY (Polygon File Format) Export
Exports voxel vertices as a point cloud or mesh structure.
- **Inputs**: A list of voxel centers or vertices.
- **Outputs**: An ASCII PLY file.
- **Steps**:
    1. Write the PLY header declaring the format as ASCII.
    2. Declare the element count (number of vertices).
    3. Declare properties: `x`, `y`, `z` (float).
    4. Write the vertex coordinates for each voxel center.
- **Note**: This treats voxels as discrete points rather than a structured grid, losing the implicit adjacency information of a voxel grid.

### VTK (Visualization Toolkit) ASCII Export
Exports voxel data as a VTK `POLYDATA` structure.
- **Inputs**: A list of voxel centers and associated scalar fields (e.g., `phase`).
- **Outputs**: A VTK 3.0 ASCII `.vtk` file.
- **Steps**:
    1. Write the VTK header (`# vtk DataFile Version 3.0`).
    2. Declare the dataset type as `POLYDATA`.
    3. Write the `POINTS` section: declare the count, then list $x, y, z$ coordinates for each voxel center.
    4. Write the `SCALARS` section: declare the name (e.g., `phase`), type (e.g., `float`), and number of components (1).
    5. Write the scalar values corresponding to each point.
- **Strengths**: High interoperability with legacy VTK-based viewers and scientific visualization tools.

### Binary Volume File Format (Image Slices)
A custom binary format for storing volumes generated from 2D image slices.
- **Inputs**: A directory of image slices, grid dimensions, and bounding box parameters.
- **Outputs**: A binary file.
- **Data Structures**:
    - **Header**: 24 bytes.
        - Bytes 0-3: `grid_dim[0]` (width, `int32`).
        - Bytes 4-7: `grid_dim[1]` (height, `int32`).
        - Bytes 8-11: `grid_dim[2]` (depth/slices, `int32`).
        - Bytes 12-15: `bbox_size[0]` (X size, `float32`).
        - Bytes 16-19: `bbox_size[1]` (Y size, `float32`).
        - Bytes 20-23: `bbox_size[2]` (Z size, `float32`).
    - **Payload**: `width * height * depth` floats (`float32`), stored in row-major order (x varying fastest, then y, then z).
- **Steps**:
    1. Read input slices from a directory, sorted by filename.
    2. Determine grid dimensions: $w, h$ from the first image shape; $d = \text{len(slices\_files)}$.
    3. Calculate bounding box size:
       $$
       \begin{aligned}
       \text{bbox\_size}[0] &= \text{BBOX\_SIZE\_X} \\
       \text{bbox\_size}[1] &= \left(\frac{\text{BBOX\_SIZE\_X}}{\text{grid\_dim}[0]}\right) \times \left(\frac{\text{vol\_size}[1]}{\text{vol\_size}[0]}\right) \times \text{grid\_dim}[1] \\
       \text{bbox\_size}[2] &= \left(\frac{\text{BBOX\_SIZE\_X}}{\text{grid\_dim}[0]}\right) \times \left(\frac{\text{vol\_size}[2]}{\text{vol\_size}[0]}\right) \times \text{grid\_dim}[2]
       \end{aligned}
       $$
       Where `BBOX_SIZE_X` is a fixed constant of `5.0` [S4, unofficial].
    4. Write the 24-byte header.
    5. Write the voxel values as a contiguous block of floats.
- **Parameters**: `BBOX_SIZE_X` = 5.0 [S4, unofficial].

### Voxel Interchange Format Serialization
Serializes a 3D voxel region into a flat 1D array with metadata, typically for JSON transport.
- **Inputs**: A voxel chunk or region defined by start coordinates and dimensions.
- **Outputs**: A JSON-like structure containing a packed array.
- **Data Structures**:
    - `voxels`: `Int8Array` (or similar byte array) of size `length * width * depth`.
    - `dimensions`: `[length, width, height]`.
    - `position`: `[startX, startY, startZ]`.
- **Steps**:
    1. Initialize `length`, `width`, `depth`, and `start` coordinates.
    2. Create an output array of size `length * width * depth`.
    3. Initialize index `idx = 0`.
    4. Loop `z` from `start[2]` to `start[2] + depth`.
    5. Loop `y` from `start[1]` to `start[1] + height`.
    6. Loop `x` from `start[0]` to `start[0] + length`.
    7. Call `getVoxelDataFor(x, y, z)` and assign to `voxels[idx]`.
    8. Increment `idx`.
- **Pitfall**: The loop bounds in some implementations mix absolute world coordinates with relative dimensions, potentially causing out-of-bounds access if not adapted to local chunk coordinates [S5, unofficial].

### PFM (Portable FloatMap) Parsing
Imports 2D or 3D volumetric data stored as PFM images.
- **Inputs**: A PFM file path.
- **Outputs**: A 3D voxel array.
- **Steps**:
    1. Read the header to determine color format (`PF` for grayscale, `Pf` for color) and endianness.
    2. Parse dimensions (width, height, and potentially depth if stacked).
    3. Read the binary float data, handling endianness conversion if necessary.
- **Note**: PFM is primarily a 2D format; 3D volumes typically require stacking multiple PFM files or a custom extension [S2, unofficial].

### ArrayMesh Byte Reader (Godot)
A specific implementation detail for reading voxel/mesh data in the Godot engine.
- **Operation**: Narrows a listed value of `-1` to `255` when reading bytes from an `ArrayMesh` structure.
- **Context**: This is a variant of byte parsing used in Godot's `variant_parser.cpp` to handle specific encoding quirks [S3, unofficial].

### Binary Volume Serialization (Bincode)
Uses the `bincode` serialization library for compact binary storage.
- **Inputs**: A Rust struct or data structure representing the voxel grid.
- **Outputs**: A binary file.
- **Steps**:
    1. Serialize the voxel data structure using `bincode`.
    2. Write the resulting bytes to a file.
- **Strengths**: Compact binary format, automatic handling of Rust types, and efficient serialization [S1, unofficial].

## Subtopics

### [Binvox File Parsing](2862-binvox-file-parsing.md)
Binvox File Parsing is the method for importing the binary voxel (binvox) file format into a contiguous array or a Sparse Voxel Octree (SVO) structure. It converts a legacy run-length encoded (RLE) voxel representation into a format suitable for modern ray-casting or rasterization engines [G]. Unlike raw voxel formats that store every voxel explicitly, binvox uses a compact binary RLE scheme, requiring specific handling to decompress the data correctly.

### [IMS File Format & Image Stack Class](3322-ims-file-format-image-stack-class.md)
The IMS file format and `ims_base` class provide a mechanism for persisting and managing 3D image stacks (volumes) with typed pixel storage and extensible metadata segments. It solves the problem of storing volumetric data where pixel types are fixed at compile time via templates and metadata is stored as typed binary segments. It differs from generic binary voxel formats by embedding a rigid 16-byte header, supporting memory-mapped loading on Windows, and using a chunked write strategy to handle large volumes without buffer overflow.

### [NRRD-like Volume I/O](4225-nrrd-like-volume-i-o.md)
This method defines a minimal, custom binary volume file format designed for efficient storage and transfer of voxel grids. It solves the problem of serializing 3D volumetric data by using a simple two-line header followed by a raw binary payload. Unlike the full Network Common Data Form (NRRD) specification, which supports complex metadata, encodings, and transformations, this variant is restricted to fixed dimensions and a single data type (`uint32_t`), prioritizing implementation simplicity and parsing speed over flexibility. It serves as a lightweight alternative to more verbose formats like [VTK Legacy ASCII File Writing](6698-vtk-legacy-ascii-file-writing.md) or the more complex [NRRD File Parsing and Volume Initialization](7216-nrrd-file-parsing-and-volume-initialization.md).

### [VTK Legacy ASCII File Writing](6698-vtk-legacy-ascii-file-writing.md)
Exports voxel data to a legacy VTK ASCII `UNSTRUCTURED_GRID` file format, representing each voxel as a hexahedral cell with associated scalar fields. It differs from binary voxel formats (e.g., [Binvox File Parsing](2862-binvox-file-parsing.md)) by using human-readable text and explicit vertex definitions for every cell, prioritizing interoperability with legacy VTK-based viewers over storage efficiency.

### [VRDF File Format / VRDF Loader](6906-vrdf-file-format-vrdf-loader.md)
The VRDF (Volume Rendering Data Format) is a binary file format designed for storing 3D voxel grids alongside associated metadata and transfer functions. It solves the problem of interoperability between medical imaging pipelines and volume rendering engines by packaging geometry, intensity data, and rendering parameters into a single self-describing file. Unlike [Binvox File Parsing](2862-binvox-file-parsing.md), which is limited to binary occupancy grids, VRDF supports continuous floating-point data, multi-channel interleaved data (e.g., label and weight), and embedded JSON metadata. It builds on [Voxel File I/O](2786-voxel-file-i-o.md) by providing a structured schema for complex volume data rather than simple raw bytes.

### PFM File Parsing
Reads a PFM (Portable FloatMap) image file, handling endianness and color format. It reads the header to determine color (PF/Pf) and dimensions, then parses the binary float data. This is primarily a 2D format, often used for storing individual slices of a volume or 2D textures within a volume pipeline [S2, unofficial].

### ArrayMesh Byte Reader
Narrows a listed -1 to 255 in the ArrayMesh byte reader, as Godot does (variant_parser.cpp:650). This is a specific implementation detail for handling byte values in Godot's mesh data structures [S3, unofficial].

### Binary Volume File Format
A binary file format for storing volume data generated from image slices, containing grid dimensions, bounding box size, and voxel values. It reads slices from an input directory, determines grid dimensions and bounding box size (with a fixed X bounding box of 5.0), and writes the data in a specific binary layout: 12 bytes for grid dimensions, 12 bytes for bounding box floats, followed by row-major voxel floats [S4, unofficial].

### Voxel Interchange Format Serialization
Serializes a 3D voxel region into a flat 1D array with metadata for transport or storage. It initializes dimensions and start coordinates, loops through the voxel grid to populate a packed array, and outputs a structure containing the voxels, dimensions, and position. A key pitfall is that loop bounds may mix absolute and relative coordinates, risking out-of-bounds access if not adapted to local chunk coordinates [S5, unofficial].

## Operating characteristics
- **CSV**: Extremely high memory and disk usage due to text overhead. Parsing speed is slow. Suitable only for small debug datasets or interoperability with spreadsheet tools.
- **PLY**: High overhead for volumetric data as it stores explicit vertices. Not suitable for dense voxel grids where implicit adjacency is preferred.
- **VTK ASCII**: High storage overhead. Parsing is slow due to text processing. Best for debugging and legacy tool compatibility.
- **Binary Volume (Custom)**: Fast I/O due to binary layout. Memory efficient for dense grids. Requires custom parsers. The fixed bounding box calculation assumes a specific aspect ratio relationship between the input image and the output volume [S4, unofficial].
- **Voxel Interchange (JSON/Array)**: Moderate overhead due to JSON metadata. The flat 1D array allows for efficient bulk transfer but requires careful index management to avoid out-of-bounds errors [S5, unofficial].
- **PFM**: Efficient for 2D slices. Endianness handling is required for cross-platform compatibility. Not natively 3D.
- **Bincode**: Very compact and fast for Rust-based pipelines. Not cross-language compatible without a corresponding deserializer.

## Background
Voxel file I/O methods have evolved alongside volume rendering techniques. Early methods relied on ASCII formats like VTK and PLY for their human-readability and wide support in scientific visualization tools [S1, unofficial]. As voxel resolutions increased, binary formats like the custom Binary Volume File Format and Binvox became necessary to manage storage and I/O performance [S4, S2, unofficial]. More recent formats like VRDF and IMS aim to support complex metadata and transfer functions required by modern neural radiance field and medical imaging pipelines [S1, S5, unofficial].

## Sources
[S1] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S2] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S3] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S4] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S5] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
