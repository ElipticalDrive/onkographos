# Voxel File I/O

Also called: Output Format Handling, Voxel Export

## Overview
Voxel File I/O covers methods for serializing and deserializing 3D voxel grids to and from persistent storage formats [S1, S4, S5, unofficial]. It handles various layouts including ASCII text (CSV, PLY, VTK), compact binary formats (BIN, custom slice-based), and interchange structures (Voxel Interchange Format, PFM) [S1, S2, S4, S5, unofficial].

## How it works
The method varies by target format, with each format defining specific inputs, outputs, data layouts, and step sequences.

### CSV Export
- **Inputs**: Voxel coordinates $(x, y, z)$ and an associated scalar value (e.g., `phase`) [S1, unofficial].
- **Outputs**: `.csv` file [S1, unofficial].
- **Steps**:
  1. Write header row: `x,y,z,phase` [S1, unofficial].
  2. Iterate through each voxel [S1, unofficial].
  3. Write row containing coordinates and scalar value [S1, unofficial].

### PLY Export
- **Inputs**: Voxel centers or vertices [S1, unofficial].
- **Outputs**: ASCII PLY file [S1, unofficial].
- **Steps**:
  1. Write ASCII PLY header with vertex count and properties `x,y,z` [S1, unofficial].
  2. Write vertex coordinates [S1, unofficial].

### VTK ASCII Export
- **Inputs**: Voxel centers and scalar fields (e.g., `phase`) [S1, unofficial].
- **Outputs**: VTK 3.0 ASCII `.vtk` file [S1, unofficial].
- **Steps**:
  1. Write VTK 3.0 ASCII POLYDATA header with `POINTS` and `SCALARS phase` [S1, unofficial].
  2. Write point coordinates and phase scalars [S1, unofficial].

### Binary Volume File Format (Image Slices)
- **Inputs**: Directory of image slices, grid dimensions, and bounding box parameters [S4, unofficial].
- **Outputs**: Binary file [S4, unofficial].
- **Data Structures**:
  - **Header** (24 bytes): `grid_dim[0]`, `grid_dim[1]`, `grid_dim[2]` (3 × `int32`), followed by `bbox_size[0]`, `bbox_size[1]`, `bbox_size[2]` (3 × `float32`) [S4, unofficial].
  - **Payload**: `width * height * depth` `float32` values in row-major order (x varying fastest, then y, then z) [S4, unofficial].
- **Steps**:
  1. Read slices from input directory, sorted by filename [S4, unofficial].
  2. Determine grid dimensions: $w, h$ from first image shape; $d = \text{len(slices\_files)}$ [S4, unofficial].
  3. Calculate bounding box size:
     $$
     \begin{aligned}
     \text{bbox\_size}[0] &= \text{BBOX\_SIZE\_X} \\
     \text{bbox\_size}[1] &= \left(\frac{\text{BBOX\_SIZE\_X}}{\text{grid\_dim}[0]}\right) \times \left(\frac{\text{vol\_size}[1]}{\text{vol\_size}[0]}\right) \times \text{grid\_dim}[1] \\
     \text{bbox\_size}[2] &= \left(\frac{\text{BBOX\_SIZE\_X}}{\text{grid\_dim}[0]}\right) \times \left(\frac{\text{vol\_size}[2]}{\text{vol\_size}[0]}\right) \times \text{grid\_dim}[2]
     \end{aligned}
     $$
     Where `BBOX_SIZE_X` is a fixed constant of `5.0` [S4, unofficial].
  4. Write grid dimensions as integers to file [S4, unofficial].
  5. Write bounding box floats [S4, unofficial].
  6. Write voxel values as `float32` in row-major order [S4, unofficial].
- **Parameters**: `BBOX_SIZE_X` = 5.0 [S4, unofficial].
- **Pseudocode**:
  ```python
  import struct
  import os

  def write_binary_volume_file(slices_dir, bbox_size_x=5.0):
      slices_files = sorted(os.listdir(slices_dir))
      first_img = load_image(slices_files[0])  # Source does not specify exact image loading function [G]
      grid_dim = [first_img.shape[0], first_img.shape[1], len(slices_files)]
      vol_size = [first_img.shape[0], first_img.shape[1], first_img.shape[2]]  # Assumes 3D input or derived dimensions [G]
      
      bbox_size = [0.0, 0.0, 0.0]
      bbox_size[0] = bbox_size_x
      bbox_size[1] = (bbox_size_x / grid_dim[0]) * (vol_size[1] / vol_size[0]) * grid_dim[1]
      bbox_size[2] = (bbox_size_x / grid_dim[0]) * (vol_size[2] / vol_size[0]) * grid_dim[2]
      
      with open("volume.bin", "wb") as f:
          f.write(struct.pack("iii", *grid_dim))
          f.write(struct.pack("fff", *bbox_size))
          for z in range(grid_dim[2]):
              for y in range(grid_dim[1]):
                  for x in range(grid_dim[0]):
                      val = read_voxel_value(x, y, z)  # Source does not specify exact slice-reading function [G]
                      f.write(struct.pack("f", val))
  ```

### Voxel Interchange Format Serialization
- **Inputs**: Voxel chunk or region defined by start coordinates and dimensions [S5, unofficial].
- **Outputs**: JSON-like structure `{ "voxels": [...], "dimensions": [...], "position": [...] }` [S5, unofficial].
- **Data Structures**: `voxels` array of size `length * width * depth` [S5, unofficial].
- **Steps**:
  1. Initialize `length`, `width`, `depth`, and `start` coordinates [S5, unofficial].
  2. Create `Int8Array` of size `length * width * depth` [S5, unofficial].
  3. Initialize `idx = 0` [S5, unofficial].
  4. Loop `z` from `start[2]` to `depth`, `y` from `start[1]` to `height`, `x` from `start[0]` to `length` [S5, unofficial].
  5. Call `getVoxelDataFor(x+start[0], y+start[1], z+start[2])` and assign to `voxels[idx]` [S5, unofficial].
  6. Increment `idx` [S5, unofficial].
- **Pitfall**: The provided loop bounds use `depth` and `height` variables that are not defined in the snippet, and `z`/`y`/`x` loop bounds mix absolute world coordinates with relative dimensions, which may cause out-of-bounds access or incorrect indexing if not adapted to local chunk coordinates [S5, unofficial].

### PFM File Parsing
- **Inputs**: PFM file path [S2, unofficial].
- **Outputs**: Voxel array [S2, unofficial].
- **Steps**:
  1. Read header to determine color format (`PF`/`Pf`) and dimensions [S2, unofficial].
  2. Handle endianness and color format during read [S2, unofficial].

### ArrayMesh Byte Reader
- **Operation**: Narrows a listed `-1` to `255` when reading bytes [S3, unofficial].
- **Context**: Implemented in Godot's `variant_parser.cpp` [S3, unofficial].

### Binary Volume Serialization (Bincode)
- **Inputs**: Voxel data structure [S1, unofficial].
- **Outputs**: Binary file [S1, unofficial].
- **Steps**: Uses `bincode` serialization for compact binary format [S1, unofficial].

## Subtopics

### [Binvox File Parsing](2862-binvox-file-parsing.md)
Binvox File Parsing is the method for importing the binary voxel (binvox) file format into a contiguous array or a Sparse Voxel Octree (SVO) structure. It converts a legacy run-length encoded (RLE) voxel representation into a format suitable for modern ray-casting or rasterization engines [G]. Unlike raw voxel formats that store every voxel explicitly, binvox uses a compact binary RLE scheme, requiring specific handling to decompress the data correctly.

### [IMS File Format & Image Stack Class](3322-ims-file-format-image-stack-class.md)
The IMS file format and `ims_base` class provide a mechanism for persisting and managing 3D image stacks (volumes) with typed pixel storage and extensible metadata segments. It solves the problem of storing volumetric data where pixel types are fixed at compile time via templates and metadata is stored as typed binary segments. It differs from generic binary voxel formats by embedding a rigid 16-byte header, supporting memory-mapped loading on Windows, and using a chunked write strategy to handle large volumes without buffer overflow.

### [NRRD-like Volume I/O](4225-nrrd-like-volume-i-o.md)
This method exports and imports volumes in a simple NRRD-like format consisting of two header lines followed by a raw binary payload. It serializes 3D voxel grids by restricting the format to fixed dimensions and a single `uint32_t` data type, prioritizing implementation simplicity and parsing speed over the flexibility of the full NRRD specification. It serves as a lightweight alternative to more verbose formats like [VTK Legacy ASCII File Writing](6698-vtk-legacy-ascii-file-writing.md) or the metadata-rich [NRRD File Parsing and Volume Initialization](7216-nrrd-file-parsing-and-volume-initialization.md).

### [VTK Legacy ASCII File Writing](6698-vtk-legacy-ascii-file-writing.md)
Exports voxel data to a legacy VTK ASCII `UNSTRUCTURED_GRID` file format, representing each voxel as a hexahedral cell with associated scalar fields. It differs from binary voxel formats (e.g., [Binvox File Parsing](2862-binvox-file-parsing.md)) by using human-readable text and explicit vertex definitions for every cell, prioritizing interoperability with legacy VTK-based viewers over storage efficiency.

### [VRDF File Format / VRDF Loader](6906-vrdf-file-format-vrdf-loader.md)
The VRDF (Volume Rendering Data Format) is a binary file format designed for storing 3D voxel grids alongside associated metadata and transfer functions. It supports continuous floating-point data, multi-channel interleaved data (e.g., label and weight), and embedded JSON metadata.

### PFM File Parsing
Reads a PFM (Portable FloatMap) image file, handling endianness and color format. It reads the header to determine color (PF/Pf) and dimensions, then parses the binary float data [S2, unofficial].

### ArrayMesh Byte Reader
Narrows a listed -1 to 255 in the ArrayMesh byte reader, as Godot does (variant_parser.cpp:650) [S3, unofficial].

### Binary Volume File Format
A binary file format for storing volume data generated from image slices, containing grid dimensions, bounding box size, and voxel values. It reads slices from an input directory, determines grid dimensions and bounding box size (with a fixed X bounding box of 5.0), and writes the data in a specific binary layout: 12 bytes for grid dimensions, 12 bytes for bounding box floats, followed by row-major voxel floats [S4, unofficial].

### Voxel Interchange Format Serialization
Serializes a 3D voxel region into a flat 1D array with metadata for transport or storage. It initializes dimensions and start coordinates, loops through the voxel grid to populate a packed array, and outputs a structure containing the voxels, dimensions, and position. A key pitfall is that loop bounds may mix absolute and relative coordinates, risking out-of-bounds access if not adapted to local chunk coordinates [S5, unofficial].

## Operating characteristics
- **CSV**: Text overhead and lack of spatial locality make it inefficient for large volumes [S1, unofficial].
- **PLY**: Stores explicit vertices, treating voxels as discrete points rather than a structured grid [S1, unofficial].
- **VTK ASCII**: High storage overhead and slow parsing due to text processing [S1, unofficial].
- **Binary Volume (Custom)**: Fast I/O due to binary layout; requires custom parsers and assumes a specific aspect ratio relationship between input image and output volume [S4, unofficial].
- **Voxel Interchange**: Flat 1D array requires careful index management to avoid out-of-bounds errors [S5, unofficial].
- **PFM**: Requires endianness handling for cross-platform compatibility [S2, unofficial].
- **ArrayMesh**: Narrows listed `-1` to `255` during byte reading [S3, unofficial].
- **Bincode**: Provides compact binary format for Rust-based pipelines [S1, unofficial].

## Background
The method is implemented across various open-source projects: voxelizer-rs provides CSV, PLY, VTK, and BIN exports [S1, unofficial]; a CUDA-based volume renderer implements a custom binary format from image slices [S4, unofficial]; a voxel engine stack provides interchange serialization [S5, unofficial]; a NeRF visualization tool includes PFM parsing [S2, unofficial]; and a renderer PR documents ArrayMesh byte handling in Godot [S3, unofficial].

## Sources
[S1] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S2] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S3] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S4] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S5] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
