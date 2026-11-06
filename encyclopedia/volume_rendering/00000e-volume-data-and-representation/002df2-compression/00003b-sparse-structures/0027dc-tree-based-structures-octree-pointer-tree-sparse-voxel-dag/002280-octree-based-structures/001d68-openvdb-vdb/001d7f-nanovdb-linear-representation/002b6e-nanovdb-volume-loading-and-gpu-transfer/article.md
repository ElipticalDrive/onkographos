# NanoVDB Volume Loading and GPU Transfer

Also called: NanoVDB Grid Reading, NanoVDB IO and Transfer

## Overview
This method describes the process of reading a NanoVDB volume file from disk, validating its data type, and transferring the raw voxel data to GPU memory for rendering. It solves the problem of bridging the gap between the CPU-side NanoVDB linear representation and the GPU-side compute buffers required by ray marching or splatting kernels. It is distinct from general VDB loading because it operates on the linear, contiguous memory layout defined in [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md), allowing for direct `memcpy`-style transfers without complex tree traversal on the CPU.

## How it works
The method assumes the input file is a valid NanoVDB file containing a grid with `float` value type. The process involves parsing the file header to extract metadata, validating the data type, allocating host memory, and uploading the data to a GPU compute buffer.

### Inputs and Outputs
- **Input**: A file path string pointing to a `.nvdb` file.
- **Output**: A GPU-ready `ComputeBuffer` (or equivalent) containing the raw voxel data, and a host-side metadata structure (`NanoVolume`) describing the buffer layout.

### Data Structures

**Host-side `NanoVolume` struct (C++/C# compatible layout):**
The structure holds metadata required to interpret the raw buffer.
- `buf`: Pointer to the raw voxel data (`uint32_t*` in C++, `uint*` in C#).
- `byteSize`: Total size of the data in bytes (`uint64_t` / `ulong`).
- `elementCount`: Number of elements in the buffer (`uint64_t` / `ulong`).
- `structStride`: Stride between elements in bytes (`uint64_t` / `ulong`).

**GPU-side Buffer:**
- Type: `ComputeBufferType.Default` (Unity) or equivalent raw buffer.
- Data: Raw `float` values, stored as `uint32_t` in memory for direct casting.

### Steps

1. **Read Grid Handle**: Call `nanovdb::io::readGrid(path)` with the file path to obtain a `GridHandle` object [S1, unofficial].
2. **Retrieve Grid Pointer**: Cast the grid handle to a `FloatGrid` using `gridHandle.grid<float>()` [S1, unofficial].
3. **Validate Buffer**: Check if the returned buffer pointer is null. If null, throw a `std::runtime_error` with the message "File did not contain a grid with value type float" [S1, unofficial].
4. **Calculate Metadata**:
   - Get `byteSize` from `gridHandle.buffer().bufferSize()` [S1, unofficial].
   - Calculate `elementCount` as `byteSize / sizeof(float)` [S1, unofficial].
   - Set `structStride` to `sizeof(float)` [S1, unofficial].
5. **Allocate Host Structure**: Allocate memory for the `NanoVolume` struct using `malloc(sizeof(struct NanoVolume))` [S1, unofficial].
6. **Assign Buffer Pointer**: Cast the grid buffer data to `uint32_t*` and assign it to `(*volume)->buf` [S1, unofficial].
7. **Assign Metadata**: Assign `byteSize`, `elementCount`, and `structStride` to the corresponding fields in the `NanoVolume` struct [S1, unofficial].
8. **Prepare for GPU Transfer (C# Specific)**:
   - Allocate a `uint[]` array `buf` on the host with size `bufferSize` (cast from `nanoVolume->elementCount`) [S1, unofficial].
   - Copy elements from the native `nanoVolume->buf` to the C# `buf` array using a loop from `i = 0` to `bufferSize` [S1, unofficial].
9. **Create GPU Buffer**: Create a Unity `ComputeBuffer` with size `bufferSize`, stride `stride` (cast from `nanoVolume->structStride`), and type `ComputeBufferType.Default` [S1, unofficial].
10. **Upload Data**: Upload the data to the GPU buffer using `gpuBuffer.SetData(buf)` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for loading NanoVDB and transferring to GPU
# Note: This combines C++ and C# steps as described in the source [S1, unofficial]

def load_nanovdb_to_gpu(file_path):
    # Step 1: Read Grid Handle
    grid_handle = nanovdb_io_read_grid(file_path)
    
    # Step 2: Retrieve Grid Pointer
    float_grid = grid_handle.get_grid_float()
    
    # Step 3: Validate Buffer
    if float_grid.buffer is None:
        raise RuntimeError("File did not contain a grid with value type float")
    
    # Step 4: Calculate Metadata
    byte_size = grid_handle.buffer.buffer_size()
    element_count = byte_size / sizeof(float)
    struct_stride = sizeof(float)
    
    # Step 5: Allocate Host Structure
    nano_volume = malloc(sizeof(NanoVolume))
    
    # Step 6: Assign Buffer Pointer (Cast to uint32_t*)
    nano_volume.buf = cast_uint32_ptr(float_grid.buffer.data)
    
    # Step 7: Assign Metadata
    nano_volume.byte_size = byte_size
    nano_volume.element_count = element_count
    nano_volume.struct_stride = struct_stride
    
    # Step 8: Prepare for GPU Transfer (C# Host Copy)
    # Allocate host array for transfer
    host_buf = allocate_uint_array(size=element_count)
    
    # Copy data from native buffer to host array
    for i from 0 to element_count - 1:
        host_buf[i] = nano_volume.buf[i]
    
    # Step 9: Create GPU Buffer
    gpu_buffer = create_compute_buffer(
        size=element_count,
        stride=struct_stride,
        type=ComputeBufferType.Default
    )
    
    # Step 10: Upload Data
    gpu_buffer.set_data(host_buf)
    
    return gpu_buffer, nano_volume
```

### Parameters
- **Value Type**: The grid is expected to have value type `float` [S1, unofficial].

### Edge Cases and Pitfalls
- **Invalid Data Type**: If the file does not contain a grid with value type `float`, the loader throws an exception and fails to load [S1, unofficial].
- **Null Buffer**: If the buffer pointer is null after retrieval, the same exception is thrown [S1, unofficial].
- **Memory Alignment**: The `structStride` is set to `sizeof(float)`, implying no padding between elements. This assumes the NanoVDB linear representation stores floats contiguously [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of voxels, dominated by the file I/O and the host-to-GPU memory copy.
- **Memory Complexity**: $O(N)$ for the host buffer during transfer, plus $O(N)$ for the GPU buffer. The `NanoVolume` struct is $O(1)$.
- **Strengths**: Simple, direct memory transfer. No complex tree traversal on the CPU. Leverages the linear layout of NanoVDB.
- **Limitations**: Requires the entire volume to fit in memory. The C# specific step involves an extra host-to-host copy, which may be optimized in other implementations.
- **When to use**: When rendering large sparse volumes on the GPU using NanoVDB's linear representation.
- **When not to use**: When memory is constrained and streaming is required, or when the volume data is not in `float` format.

## Background
NanoVDB is a GPU-optimized implementation of OpenVDB, designed for efficient storage and access on the GPU. The linear representation allows for compact copying via `memcpy`. This specific loading and transfer method is derived from the Unity NanoVDB renderer project, which provides an unofficial implementation for integrating NanoVDB volumes into Unity's compute shader pipeline [S1, unofficial].

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
