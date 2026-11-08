# Compute Shader Render Pass

Also called: Compute Shader Pass

## Overview
A Compute Shader Render Pass is a rendering technique that bypasses the traditional fixed-function graphics pipeline by using a compute shader to write directly to a framebuffer (or storage buffer acting as a framebuffer) [S1, unofficial]. It solves the problem of rendering complex or non-standard primitives (such as point clouds or splats) where hardware rasterization is inefficient or insufficient [S1, unofficial]. Unlike standard rasterization, which relies on vertex processing and primitive assembly, this method maps computational tasks directly to pixels or fragments via parallel execution units, allowing for custom visibility resolution and shading logic [G]. It differs from [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by often operating on a per-pixel or per-fragment basis without explicit tile-based optimization, and from [Compute Shader Kernel](1202-compute-shader-kernel.md) by specifically targeting image output and integration with the present/surface lifecycle.

## How it works
The method involves setting up a compute pipeline, binding resources, dispatching workgroups, and managing image layout transitions for presentation [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Image dimensions (width, height), source data (e.g., point cloud positions, colors, or Gaussian parameters), and current swapchain image index.
*   **Output:** A color/depth buffer (swapchain image or intermediate texture) containing the rendered result.

**Data Structures:**
*   **Push Constants:** Typically a `uint32_t[2]` array storing `{width, height}` [S1, unofficial].
*   **Storage Buffers:** Bindless or descriptor-set bound buffers containing vertex/point data and intermediate state (e.g., sort buffers, histograms) [S1, unofficial].
*   **Image Views:** The target swapchain image or render target, accessed as a storage image or color attachment [S1, unofficial].

**Steps:**
1.  **Pipeline Binding:** Bind the compute pipeline (`renderPipeline`) and its associated descriptor sets. The descriptor sets must map storage buffers to specific binding indices (e.g., index 0 for data, index 1 for histograms) [S1, unofficial].
2.  **Push Constants:** Push the image dimensions (width and height) as `uint32_t` constants to the compute shader [S1, unofficial].
3.  **Layout Transition:** Transition the target swapchain image layout from `VK_IMAGE_LAYOUT_UNDEFINED` to `VK_IMAGE_LAYOUT_GENERAL` to allow storage image access [S1, unofficial].
4.  **Dispatch:** Execute the compute shader. The workgroup dimensions are calculated to cover the image area, typically using a 16x16 thread block size:
    $$
    \text{Workgroups}_x = \lceil \frac{\text{width}}{16} \rceil, \quad \text{Workgroups}_y = \lceil \frac{\text{height}}{16} \rceil
    $$
    The dispatch command is issued as `dispatchWorkgroups(Workgroups_x, Workgroups_y, 1)` [S1, unofficial].
5.  **Final Layout Transition:**
    *   If a GUI overlay (e.g., ImGui) is enabled: Transition the image layout to `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` [S1, unofficial].
    *   Otherwise: Transition directly to `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` [S1, unofficial].
6.  **GUI Rendering (Optional):** If the GUI is enabled, draw the overlay into the image while it is in `COLOR_ATTACHMENT_OPTIMAL` layout [S1, unofficial].
7.  **Present Transition:** If a GUI was drawn, transition the image layout from `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` back to `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` [S1, unofficial].
8.  **Presentation:** Present the image to the screen [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shader_render_pass(device, command_buffer, render_pipeline, descriptor_sets, 
                               swapchain_image, width, height, gui_enabled):
    # 1. Bind Pipeline and Descriptor Sets
    vk.cmd_bind_pipeline(command_buffer, vk.PipelineType.COMPUTE, render_pipeline.handle)
    for i, desc_set in enumerate(descriptor_sets):
        vk.cmd_bind_descriptor_sets(command_buffer, vk.PipelineLayoutType.COMPUTE, 
                                    render_pipeline.layout, i, desc_set.handle, 0, None)

    # 2. Push Constants (Width, Height)
    push_constants = struct.pack('II', width, height)
    vk.cmd_push_constants(command_buffer, render_pipeline.layout, 
                          vk.ShaderStageFlagBits.COMPUTE, 0, len(push_constants), push_constants)

    # 3. Transition Image to General Layout for Compute Access
    image_barrier = vk.ImageMemoryBarrier(
        src_access_mask=vk.AccessFlags(),
        dst_access_mask=vk.AccessFlags(IMAGE_WRITE),
        old_layout=vk.ImageLayout.UNDEFINED,
        new_layout=vk.ImageLayout.GENERAL,
        src_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
        dst_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
        subresource_range=vk.ImageSubresourceRange(
            aspect_mask=vk.ImageAspectFlags.COLOR,
            base_mip_level=0,
            level_count=1,
            base_array_layer=0,
            layer_count=1
        )
    )
    vk.cmd_pipeline_barrier(command_buffer, vk.StageFlags.COMPUTE_SHADER, 
                            vk.StageFlags.COMPUTE_SHADER, 0, None, None, [image_barrier])

    # 4. Dispatch Compute Shader
    group_count_x = (width + 15) // 16
    group_count_y = (height + 15) // 16
    vk.cmd_dispatch(command_buffer, group_count_x, group_count_y, 1)

    # 5. Transition Image for Presentation or GUI
    if gui_enabled:
        # Transition to Color Attachment for GUI drawing
        gui_barrier = vk.ImageMemoryBarrier(
            src_access_mask=vk.AccessFlags(IMAGE_WRITE),
            dst_access_mask=vk.AccessFlags(DEVICE_READ),
            old_layout=vk.ImageLayout.GENERAL,
            new_layout=vk.ImageLayout.COLOR_ATTACHMENT_OPTIMAL,
            src_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
            dst_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
            subresource_range=vk.ImageSubresourceRange(
                aspect_mask=vk.ImageAspectFlags.COLOR,
                base_mip_level=0,
                level_count=1,
                base_array_layer=0,
                layer_count=1
            )
        )
        vk.cmd_pipeline_barrier(command_buffer, vk.StageFlags.COMPUTE_SHADER, 
                                vk.StageFlags.FRAGMENT_SHADER, 0, None, None, [gui_barrier])
        
        # Draw GUI (e.g., ImGui) here
        draw_gui(command_buffer, swapchain_image)

        # Transition to Present Layout
        present_barrier = vk.ImageMemoryBarrier(
            src_access_mask=vk.AccessFlags(DEVICE_READ),
            dst_access_mask=vk.AccessFlags(),
            old_layout=vk.ImageLayout.COLOR_ATTACHMENT_OPTIMAL,
            new_layout=vk.ImageLayout.PRESENT_SRC_KHR,
            src_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
            dst_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
            subresource_range=vk.ImageSubresourceRange(
                aspect_mask=vk.ImageAspectFlags.COLOR,
                base_mip_level=0,
                level_count=1,
                base_array_layer=0,
                layer_count=1
            )
        )
        vk.cmd_pipeline_barrier(command_buffer, vk.StageFlags.FRAGMENT_SHADER, 
                                vk.StageFlags.PRESENT, 0, None, None, [present_barrier])
    else:
        # Direct transition to Present Layout
        present_barrier = vk.ImageMemoryBarrier(
            src_access_mask=vk.AccessFlags(IMAGE_WRITE),
            dst_access_mask=vk.AccessFlags(),
            old_layout=vk.ImageLayout.GENERAL,
            new_layout=vk.ImageLayout.PRESENT_SRC_KHR,
            src_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
            dst_queue_family_index=vk.QUEUE_FAMILY_IGNORED,
            subresource_range=vk.ImageSubresourceRange(
                aspect_mask=vk.ImageAspectFlags.COLOR,
                base_mip_level=0,
                level_count=1,
                base_array_layer=0,
                layer_count=1
            )
        )
        vk.cmd_pipeline_barrier(command_buffer, vk.StageFlags.COMPUTE_SHADER, 
                                vk.StageFlags.PRESENT, 0, None, None, [present_barrier])
```

**Parameters:**
*   **Workgroup Size:** Typically 16x16x1 threads [S1, unofficial].
*   **Image Layouts:** `GENERAL` for compute access, `COLOR_ATTACHMENT_OPTIMAL` for GUI, `PRESENT_SRC_KHR` for presentation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in compute shaders may differ from fragment shaders; ensure consistent data types for depth/color packing if used [S1, unofficial].
*   **Boundaries:** Ensure dispatch dimensions cover the entire image, rounding up to the nearest workgroup size [S1, unofficial].
*   **Synchronization:** Memory barriers are critical to ensure compute writes are visible to subsequent GUI or present operations [S1, unofficial].

## Subtopics

### [Atomic Depth-Color Packing for Point Clouds](10206-atomic-depth-color-packing-for-point-clouds.md)
This variant renders point clouds by replacing the hardware pipeline's `GL_POINTS` primitive with a custom storage buffer acting as a framebuffer. It resolves visibility and color simultaneously by encoding depth and color into a single 64-bit integer and using 64-bit `atomicMin` to select the point with the lowest depth for each pixel. Unlike Günther et al. [GKLR13], which uses a busy-loop to wait for a pixel to be free and locks it with atomic compare-and-swap, this method updates an interleaved depth and color buffer with a single atomic instruction, improving performance.

### Buffer Reset
Uses a compute shader to reset a buffer to a specific value [S2, unofficial]. It binds a storage buffer object (SSBO) to a reset shader, sets uniform variables for the array size and target value, and executes the shader with a number of groups determined by the buffer size and maximum group size [S2, unofficial].

### Overlapped Compute-Render Scheduling
A double-buffering strategy to systematically overlap compute (LOD calculation, vertex assembly) and rendering passes by swapping front and back buffers between frames [S3]. In Frame N, the CPU submits a command buffer for compute passes writing to the back buffer, and a second command buffer for rendering using the front buffer (prepared in Frame N-1) [S3]. Synchronization is implicitly handled by the queue, ensuring commands execute in submission order [S3]. This introduces a one-frame latency (~16.7 ms) but improves effective frame rate by ~11% compared to serial execution [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and work items processed per pixel [G].
*   **Memory Complexity:** Depends on the size of the input data buffers and the framebuffer [G].
*   **Performance:** Highly dependent on GPU hardware and memory bandwidth. Efficient for irregular data distributions where rasterization is inefficient [S1, unofficial].
*   **Strengths:** Full control over rendering logic, ability to handle complex visibility resolution (e.g., atomic depth testing), and compatibility with non-standard primitives [S1, unofficial].
*   **Limitations:** Requires manual management of memory barriers and layout transitions, which can be error-prone [S1, unofficial]. Lack of hardware rasterization optimizations (e.g., early-Z) unless explicitly implemented [G].
*   **When to Use:** Point clouds, splats, or custom rendering algorithms where the fixed-function pipeline is insufficient [S1, unofficial].
*   **When Not to Use:** Simple geometric primitives where hardware rasterization is more efficient [G].

## Background
The technique is rooted in General-Purpose computing on Graphics Processing Units (GPGPU) and has been adapted for rendering tasks in modern graphics APIs like Vulkan and WebGPU [S1, unofficial][S3]. It allows for flexible, data-parallel rendering pipelines that can be integrated with traditional graphics passes [S3].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S3] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
