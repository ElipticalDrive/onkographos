# Volume Upload via Staging Buffer

Also called: GPU Memory Transfer via Staging, Staging Buffer Upload

## Overview
This method transfers dense 3D volume data from host memory to GPU device-local memory using an intermediate staging buffer [S1, unofficial]. It solves the problem of populating a GPU-resident 3D texture when direct host-to-device mapping is unsupported or when data preparation is required before upload [S1, unofficial]. It is distinct from [Device Volume Snapshot](2246-device-volume-snapshot.md), which captures existing device data, and [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md), which may use unified memory architectures [G].

## How it works
**Inputs:**
- `dims`: Volume dimensions `nx`, `ny`, `nz` [S1, unofficial].
- `field`: Host-side array of volume data (typically `float`) [S1, unofficial].
- `kChannels`: Number of channels per voxel [S1, unofficial].
- `kVolumeFormat`: GPU texture format (e.g., `R32F`, `RGBA32F`) [S1, unofficial].

**Outputs:**
- A GPU 3D image resource bound to a 3D image view, ready for sampling [S1, unofficial].

**Data Structures:**
1. **Volume Dimensions (`dims`)**:
   - `nx`: Width, `ny`: Height, `nz`: Depth [S1, unofficial].
   - `count()`: Returns `nx * ny * nz` [G].
2. **Staging Buffer**:
   - Host-visible, host-coherent memory allocation [S1, unofficial].
   - Size: `byteSize` bytes [S1, unofficial].
   - Creation info: size `byteSize`, usage `eTransferSrc`, sharing mode `eExclusive` [S1, unofficial].
3. **Device Image**:
   - Creation info: type `e3D`, format `kVolumeFormat`, extent `(dims.nx, dims.ny, dims.nz)`, mip levels `1`, array layers `1`, samples `e1`, tiling `eOptimal`, usage `eSampled | eTransferDst | eTransferSrc`, initial layout `eUndefined` [S1, unofficial].
   - Allocated in device-local memory [S1, unofficial].
4. **Image View**:
   - Creation info: image handle, view type `e3D`, format `kVolumeFormat`, subresource range aspect `eColor`, base mip level `0`, level count `1`, base array layer `0`, layer count `1` [S1, unofficial].

**Steps:**
1. **Validate Dimensions**: Check if `dims.nx`, `dims.ny`, or `dims.nz` exceed the device's `limits.maxImageDimension3D`. If any dimension exceeds the limit, return error `Errc::unsupported_configuration` [S1, unofficial].
2. **Calculate Size**: Compute the required byte size for the staging buffer:
   $$ \text{byteSize} = \text{dims.count()} \times \text{kChannels} \times \text{sizeof(float)} $$
   [S1, unofficial]
3. **Allocate Resources**: Allocate device-local memory for the 3D image and host-visible, host-coherent memory for the staging buffer [S1, unofficial].
4. **Populate Staging Buffer**: Map the staging memory, create a `std::span<float>` of size `dims.count() * kChannels`, and call `deriveField<T>` to populate it with field data, returning `autoRange` [S1, unofficial].
5. **Create GPU Resources**: Create the 3D image and image view using the creation info defined in **Data Structures** [S1, unofficial].
6. **Upload Data**: Record a command buffer command to copy the data from the staging buffer (`eTransferSrc`) to the 3D image (`eTransferDst`), execute the command buffer, and wait for completion [G].
7. **Cleanup**: Destroy the staging buffer and its memory, then destroy the image view, image, and their respective memory allocations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_volume_to_gpu(field, dims, k_channels, k_volume_format):
    # 1. Validate dimensions
    max_dim = get_device_limit("maxImageDimension3D")
    if dims.nx > max_dim or dims.ny > max_dim or dims.nz > max_dim:
        raise ValueError("Errc::unsupported_configuration")

    # 2. Calculate size
    voxel_count = dims.nx * dims.ny * dims.nz
    byte_size = voxel_count * k_channels * 4  # sizeof(float)

    # 3. Allocate resources
    staging_buffer = allocate_buffer(size=byte_size, usage="eTransferSrc", sharing_mode="eExclusive")
    staging_memory = allocate_memory(buffer=staging_buffer, properties="eHostVisible | eHostCoherent")
    bind_memory(buffer=staging_buffer, memory=staging_memory)

    gpu_image = create_image(type="e3D", format=k_volume_format, extent=(dims.nx, dims.ny, dims.nz),
                             mip_levels=1, array_layers=1, samples="e1", tiling="eOptimal",
                             usage="eSampled | eTransferDst | eTransferSrc", initial_layout="eUndefined")
    image_memory = allocate_memory(image=gpu_image, properties="eDeviceLocal")
    bind_memory(image=gpu_image, memory=image_memory)

    # 4. Populate staging buffer
    mapped_data = map_memory(staging_memory)
    span = mapped_data[:voxel_count * k_channels]
    auto_range = derive_field(span, field)
    unmap_memory(staging_memory)

    # 5. Create image view
    image_view = create_image_view(image=gpu_image, view_type="e3D", format=k_volume_format,
                                   aspect="eColor", base_mip_level=0, level_count=1,
                                   base_array_layer=0, layer_count=1)

    # 6. Upload data
    command_buffer = begin_command_buffer()
    transition_layout(command_buffer, gpu_image, "eUndefined", "eTransferDstOptimal")
    copy_buffer_to_image(command_buffer, staging_buffer, gpu_image, dims)
    transition_layout(command_buffer, gpu_image, "eTransferDstOptimal", "eShaderReadOnlyOptimal")
    end_command_buffer(command_buffer)
    submit_and_wait(command_buffer)

    # 7. Cleanup staging
    destroy_buffer(staging_buffer)
    free_memory(staging_memory)

    return {"image": gpu_image, "view": image_view, "dims": dims, "auto_range": auto_range}
```

**Edge Cases and Pitfalls:**
- **Memory Destruction Order**: Memory must be destroyed in reverse order of binding to avoid freeing memory bound to destroyed objects. The required order is: `imageMem`, `image`, `view`, `stagingMem`, `staging` [S1, unofficial].
- **Dimension Limits**: Exceeding `maxImageDimension3D` results in an unsupported configuration error [S1, unofficial].
- **Coherency**: The staging buffer must be host-coherent to ensure written data is visible to the GPU without explicit cache flushes, or explicit flushes must be performed [S1, unofficial].

## Subtopics

### Volume Readback via Staging Buffer
This variant transfers data from the GPU device to the host [G]. It allocates a host-visible, host-coherent staging buffer with usage `eTransferDst` [S1, unofficial]. After rendering or processing, the GPU image is copied to the staging buffer [G]. The host then maps the staging memory, copies `byteSize` bytes into a `std::vector<float>`, unmaps the memory, and returns a `VolumeReadback` struct containing the dimensions and data [S1, unofficial]. The byte size calculation is identical: `byteSize = dims.count() * kChannels * sizeof(float)` [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by the PCIe transfer time between host and device. The copy operation is linear with respect to the number of voxels [G].
- **Memory Complexity**: Requires additional host memory equal to `byteSize` for the staging buffer [S1, unofficial].
- **Performance**: Slower than direct mapping if the hardware supports it, due to the extra copy step. However, it is often necessary for optimal performance because it allows the GPU to use device-local memory (optimized for sampling) while using host-coherent memory for efficient CPU writes [G].
- **Strengths**: Works on hardware without direct host-to-device mapping support; ensures optimal GPU memory layout (optimal tiling) [S1, unofficial].
- **Limitations**: Requires double the memory footprint during upload (host staging + device image); introduces latency due to the copy operation [G].

## Background
This method is a standard technique in GPU programming, particularly in Vulkan and OpenGL, for managing volume data [G]. It is implemented in libraries such as `imaginaryVolumes` [S1, unofficial]. The approach leverages the Vulkan memory model's separation of host-visible and device-local memory to balance CPU write efficiency and GPU read performance [G].

## Sources
[S1] steubengineer/imaginaryVolumes, https://github.com/steubengineer/imaginaryVolumes, fetched 2026-10-26 (unofficial)
