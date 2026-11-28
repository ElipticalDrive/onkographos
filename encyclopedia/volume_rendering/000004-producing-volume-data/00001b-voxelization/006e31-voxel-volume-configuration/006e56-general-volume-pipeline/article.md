# General Volume Pipeline

Also called: Allocation, Tagging, and Conversion

## Overview
The General Volume Pipeline is a voxel data management architecture designed to solve the rigidity of monolithic voxel formats by enabling dynamic attribute addition and format conversion [S1, unofficial]. It allows engines to utilize multiple voxel formats tailored to specific tasks, analogous to how graphics engines handle 3D model imports and shader conversions [S1, unofficial]. Unlike systems bound to a single static layout, this pipeline separates data allocation, attribute definition, and format transformation into distinct stages [S1, unofficial].

## How it works
The pipeline consists of three sequential stages: Allocation, Tagging, and Conversion [S1, unofficial].

### Inputs and Outputs
*   **Input**: A volume allocator context and a target memory location (e.g., CPU or GPU).
*   **Output**: A volume data structure with defined attributes, converted into a target format suitable for rendering or processing.

### Data Structures
The volume header contains an array of attributes. Each attribute is defined by the following fields [S1, unofficial]:
*   `bits_per_element`: Size in bits.
*   `type`: An enum mirroring GLSL primitives (e.g., `U8VEC4`, `VEC3`) or a 'custom' label.
*   `total_size_in_bytes`: Total memory size for the attribute across the volume.
*   `name`: A string identifier for the attribute.

### Steps

1.  **Allocation**: Create a buffer for voxel data using a volume allocator context. The developer specifies the target memory location via the allocator name (e.g., "cpu_recycled" for CPU, or a GPU-specific allocator) [S1, unofficial]. The allocator manages the buffer lifecycle, and the developer is responsible only for returning it when finished [S1, unofficial].

2.  **Tagging**: Assign attributes to the allocated volume data. For each attribute, define its name, size, data type, and optional data pointer, storing these in the volume header [S1, unofficial]. The data type must mirror GLSL primitives or be labeled 'custom' [S1, unofficial].

3.  **Conversion**: Convert voxel data from one format to another. This involves binding a conversion function to specific attribute type combinations. The conversion function is selected at runtime based on the source and target attribute types [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stage 1: Allocation
# Retrieve allocator context and create volume buffer
allocator = get_volume_allocator_from_name("cpu_recycled")
dest_volume = allocator.new_volume()

# Stage 2: Tagging
# Define Albedo attribute
albedo_attr = Attribute()
albedo_attr.set_name("albedo")
albedo_attr.bits_per_element = sizeof(u8vec4) * 8
albedo_attr.type = VOLUME_ATTRIBUTE_TYPE_U8VEC4
albedo_attr.total_size_in_bytes = aabb_volume * sizeof(u8vec4)
dest_volume.header.attributes[0] = albedo_attr

# Define Normal attribute
normal_attr = Attribute()
normal_attr.set_name("normal")
normal_attr.bits_per_element = sizeof(vec3) * 8
normal_attr.type = VOLUME_ATTRIBUTE_TYPE_VEC3
normal_attr.total_size_in_bytes = aabb_volume * sizeof(vec3)
dest_volume.header.attributes[1] = normal_attr

# Stage 3: Conversion
# Bind conversion function based on attribute types
# Note: This requires pre-coding supported type combinations at compile time
if (dest_volume.header.attributes[0].type == VOLUME_ATTRIBUTE_TYPE_U8VEC4 and
    dest_volume.header.attributes[1].type == VOLUME_ATTRIBUTE_TYPE_VEC3):
    conversion_func = bind(convert_terrain_block_to_static_standard, u8vec4, vec3)

# Execute conversion
# Iterate through voxels and write to binary writers
for x in range(aabb_size.x):
    for y in range(aabb_size.y):
        for z in range(aabb_size.z):
            src_albedo = cell_reader.read_u8vec4()
            src_normal = cell_reader.read_vec3()
            # ... conversion logic ...
            albedo_writer.write(src_albedo)
            normal_writer.write(src_normal)
```

### Parameters and Edge Cases
*   **Type Combinations**: The tagging system requires pre-coding supported type combinations because the compiler must generate template conversion functions at compile time [S1, unofficial].
*   **Runtime Flexibility**: While the function pointer can be changed at runtime, the underlying template functions must be declared [S1, unofficial].
*   **Memory Management**: The allocator handles lower-level tasks; the developer returns the buffer when finished [S1, unofficial].

## Operating characteristics
*   **Flexibility**: High. Allows dynamic attribute addition and format conversion [S1, unofficial].
*   **Complexity**: Moderate. Requires managing multiple allocators and conversion functions [S1, unofficial].
*   **Performance**: Conversion is performed at compile time for supported types, ensuring efficient runtime execution [S1, unofficial].
*   **Limitations**: Limited by the pre-coded type combinations; unsupported types require new template functions [S1, unofficial].

## Background
The concept originates from graphics programming practices used with 3D models (e.g., OBJ, PLY, FBX) and shaders, where engines import models into a common format and convert them for GPU rasterization [S1, unofficial]. It extends this philosophy to voxel data, drawing inspiration from physics and ray tracing libraries that rely on triangle data as a core common format [S1, unofficial]. The approach is part of an engine design philosophy centered on Entity Component System (ECS) principles to enable flexibility and expandability [S1, unofficial].

## Sources
[S1] The Perfect Voxel Engine, https://voxely.net/blog/the-perfect-voxel-engine/, fetched 2026-10-26 (unofficial)
