# Adjoint Intersect Raster Pass

Also called: Backward Intersect Raster Pass

## Overview
The Adjoint Intersect Raster Pass is a backward pass that computes gradients for the forward tetrahedral rasterization to enable differentiable volume rendering [S1, unofficial]. It sets apart from the forward pass by operating in a fragment shader with interlock mechanisms to safely accumulate gradient contributions, whereas the forward pass computes color accumulation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** The pass consumes the same geometric data as the forward pass: `UniformDataBuffer` (camera/view matrices), `TriangleKeyValueBuffer`, `TriangleCounterBuffer` (optional), `TriangleVertexPositionBuffer`, `TriangleTetIndexBuffer`, `TetIndexBuffer`, `TetVertexPositionBuffer`, and either `TetVertexColorBuffer` or `TetCellColorBuffer` [S1, unofficial].
- **Outputs:** Gradient buffers for the tetrahedral cell colors or vertex colors, updated via atomic additions [S1, unofficial].

**Data Structures:**
The buffers are bound statically. The `DrawIndirectBuffer` contains a single `VkDrawIndirectCommand` with size `sizeof(VkDrawIndirectCommand)` and indirect draw count 1 [S1, unofficial].

**Steps:**
1. **Extension Check:** Verify that the device supports `VK_EXT_FRAGMENT_SHADER_INTERLOCK_EXTENSION_NAME`. If unsupported, throw an error [S1, unofficial].
2. **Shader Configuration:** Load shaders with preprocessor defines `BACKWARD_PASS`, `PI_SQRT`, and `INV_PI_SQRT` [S1, unofficial].
3. **Atomic Support Check:** If the hardware feature `shaderBufferFloat32AtomicAdd` is supported, add the define `SUPPORT_BUFFER_FLOAT_ATOMIC_ADD` and set the GLSL extension string to `"GL_EXT_shader_atomic_float;GL_EXT_control_flow_attributes"` [S1, unofficial].
4. **Buffer Binding:** Bind the static buffers listed in Inputs [S1, unofficial].
5. **Pipeline Configuration:**
   - Disable culling.
   - Set front face to Counter-Clockwise (CCW).
   - Disable color write.
   - Disable depth write and depth test.
   - Set blend mode to OVERWRITE [S1, unofficial].
6. **Execution:** Issue an indirect draw call using the `DrawIndirectBuffer` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adjoint_intersect_raster_pass(device, pipeline_layout, shader_module, buffers):
    # Step 1: Check extension support
    if not device.supports_extension("VK_EXT_FRAGMENT_SHADER_INTERLOCK_EXTENSION_NAME"):
        raise RuntimeError("VK_EXT_FRAGMENT_SHADER_INTERLOCK_EXTENSION_NAME is required for Adjoint Intersect Raster Pass")

    # Step 2 & 3: Configure shader defines
    defines = ["BACKWARD_PASS", "PI_SQRT", "INV_PI_SQRT"]
    if device.features.shader_buffer_float32_atomic_add:
        defines.append("SUPPORT_BUFFER_FLOAT_ATOMIC_ADD")
        glsl_extensions = "GL_EXT_shader_atomic_float;GL_EXT_control_flow_attributes"
    else:
        glsl_extensions = ""

    # Step 4: Bind buffers
    # UniformDataBuffer, TriangleKeyValueBuffer, TriangleCounterBuffer (optional),
    # TriangleVertexPositionBuffer, TriangleTetIndexBuffer, TetIndexBuffer,
    # TetVertexPositionBuffer, TetVertexColorBuffer/TetCellColorBuffer
    bind_static_buffers(buffers)

    # Step 5: Configure pipeline state
    pipeline_state = {
        "culling": False,
        "front_face": "CCW",
        "color_write": False,
        "depth_write": False,
        "depth_test": False,
        "blend_mode": "OVERWRITE"
    }
    configure_graphics_pipeline(pipeline_state, defines, glsl_extensions)

    # Step 6: Execute indirect draw
    draw_indirect_buffer = create_buffer(size=sizeof(VkDrawIndirectCommand), count=1)
    draw_indirect(draw_indirect_buffer)
```

**Parameters:**
- `VK_EXT_FRAGMENT_SHADER_INTERLOCK_EXTENSION_NAME`: Required for fragment shader interlock to safely handle concurrent writes in the adjoint pass [S1, unofficial].
- `shaderBufferFloat32AtomicAdd`: Hardware feature flag that enables `SUPPORT_BUFFER_FLOAT_ATOMIC_ADD` and `GL_EXT_shader_atomic_float` extension for atomic float operations in the backward pass [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `shaderBufferFloat32AtomicAdd` is not supported, atomic float operations in the fragment shader will fail or fall back to slower mechanisms, potentially causing incorrect gradient accumulation [S1, unofficial].
- The pass relies on `VK_EXT_FRAGMENT_SHADER_INTERLOCK`; without it, concurrent writes to gradient buffers from multiple fragments will result in race conditions [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Enables differentiable volume rendering by computing gradients for the forward pass [S1, unofficial].
- **Limitations:** Requires specific Vulkan extensions (`VK_EXT_FRAGMENT_SHADER_INTERLOCK_EXTENSION_NAME`) and hardware features (`shaderBufferFloat32AtomicAdd`) [S1, unofficial].
- **When to use:** When implementing differentiable volume rendering with tetrahedral meshes [S1, unofficial].
- **When not to use:** When hardware does not support the required Vulkan extensions or atomic float operations [S1, unofficial].

## Background
The Adjoint Intersect Raster Pass is part of the DiffTetVR project, a differentiable volume renderer for tetrahedral meshes [S1, unofficial]. It was developed to enable gradient computation through the rasterization process, allowing for optimization of volume rendering parameters [S1, unofficial].

## Sources
[S1] GitHub - chrismile/DiffTetVR: Differentiable volume renderer for ..., https://github.com/chrismile/DiffTetVR, fetched 2026-10-28 (unofficial)
