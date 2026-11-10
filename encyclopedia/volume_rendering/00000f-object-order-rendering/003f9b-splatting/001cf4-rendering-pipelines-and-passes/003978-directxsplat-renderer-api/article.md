# DirectXSplat Renderer API

Also called: DirectXSplat, DirectXSplat Rendering Pipeline, directxsplat::Renderer

## Overview
DirectXSplat is a C++/Direct3D 12 library for rendering 3D Gaussian Splats, providing a rasterization pipeline that handles scene upload, preparation, rendering, and frame submission with explicit resource management [S1, unofficial]. It differs from general-purpose ray-marching or CPU-based splatting implementations by leveraging native D3D12 hardware acceleration for high-performance, real-time rendering, achieving up to 2.4x faster throughput than `gsplat` in matched-quality benchmarks [S1, unofficial]. It is designed for host-owned device architectures where the application manages command lists, fences, and render targets, offering fine-grained control over memory residency and upload budgets [S1, unofficial].

## How it works
The API implements a prepare-then-render pipeline. The host application initializes a `D3D12Context`, loads a scene into a `directxsplat::Scene` object, and creates an `UploadedSceneHandle` to transfer data to GPU memory [S1, unofficial]. For each frame, the host populates a `RenderFrameContext` with fence state and calls `PrepareSceneForRender` to cull, LOD-select, and stage Gaussians [S1, unofficial]. Finally, `Render` is called with a reset command list and target binding to record the rasterization commands [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- `ID3D12Device*`, `ID3D12CommandQueue*`, `ID3D12Fence*`: Host-owned D3D12 primitives [S1, unofficial].
- `directxsplat::Scene`: Loaded scene data (PLY, SPZ, .splat, SOG, or hierarchical LOD metadata) [S1, unofficial].
- `RenderInput`: Camera matrices, view parameters, and frame index [S1, unofficial].
- `ID3D12GraphicsCommandList*`: A reset, open command list for recording [S1, unofficial].
- `RenderTargetBinding`: Color (and optional depth) target resources, RTV/DSV handles, formats, and states [S1, unofficial].

**Outputs:**
- `RenderResult`: Contains submission status, sync points for uploads, and error codes [S1, unofficial].
- Final image rendered to the bound color target [S1, unofficial].

### Data Structures

**`RenderFrameContext`**
Tracks GPU synchronization state for the current frame.
- `fence`: `ID3D12Fence*` (Same fence passed to `D3D12Context::Initialize`)
- `completedFenceValue`: `uint64_t` (Latest known completed value from `fence->GetCompletedValue()`)
- `submissionFenceValue`: `uint64_t` (Unique, nonzero, strictly greater than previous signals and `completedFenceValue`)
- `frameIndex`: `uint32_t` (Host frame index)

**`RenderTargetBinding`**
Describes the render target and transition requirements.
- `colorTarget`: `ID3D12Resource*`
- `colorRtv`: `D3D12_CPU_DESCRIPTOR_HANDLE`
- `colorFormat`: `DXGI_FORMAT` (Must match RTV's single-sampled format)
- `colorStateBefore`: `D3D12_RESOURCE_STATES`
- `colorStateAfter`: `D3D12_RESOURCE_STATES`
- `transitionMode`: `ResourceTransitionMode` (`LibraryManaged` or `CallerManaged`)
- `viewport`: `D3D12_VIEWPORT`
- `scissor`: `D3D12_RECT`

**`RendererConfig`**
- `residencyBudgetGaussians`: `uint64_t` (Default: $16 \times 1024 \times 1024$)
- `uploadBudgetGaussians`: `uint64_t` (Default: 650,000)
- `warmUploadBudgetGaussians`: `uint64_t` (Default: 1,200,000)
- `maxUploadsPerFrame`: `uint32_t` (Default: 16)
- `chunkTargetSize`: `uint32_t` (Default: 65,536)
- `lod0ScreenRadius`: `float` (Default: 72.0f)
- `lod1ScreenRadius`: `float` (Default: 24.0f)
- `cullScreenRadius`: `float` (Default: 0.35f)
- `residencyCacheFrames`: `uint32_t` (Default: 120)
- `enableGpuTiming`: `bool` (Default: true)

**Gaussian Primitive Fields**
- `position`: `Vec3`
- `scale`: `Vec3`
- `rotation`: `Vec4` (Quaternion)
- `opacity`: `float`
- `sh`: `array` of floats (Spherical Harmonics coefficients)

### Numbered Steps

1.  **Initialize Context**: Create `D3D12Context` with host `ID3D12Device*`, `ID3D12CommandQueue*`, and `ID3D12Fence*` [S1, unofficial].
2.  **Initialize Renderer**: Create `directxsplat::Renderer` instance [S1, unofficial].
3.  **Load Scene**: Call `LoadSceneFromFile(scenePath)` to detect format (PLY, compressed PLY, SPZ, .splat, SOG, LOD metadata) and populate `directxsplat::Scene` [S1, unofficial].
4.  **Create Uploaded Scene**: Call `renderer.CreateUploadedScene(loadedScene, sceneHandle)` to transfer data to GPU memory [S1, unofficial].
5.  **Prepare Frame Context**: Populate `RenderFrameContext`:
    - Set `fence` to the direct queue fence.
    - Set `completedFenceValue` to `fence->GetCompletedValue()`.
    - Set `submissionFenceValue` to a unique, increasing value (nonzero, > previous signals, > `completedFenceValue`) [S1, unofficial].
    - Set `frameIndex` from input [S1, unofficial].
6.  **Prepare Scene**: Call `renderer.PrepareSceneForRender(sceneHandle, input, frameContext, &preparationResult)` [S1, unofficial].
7.  **Configure Target**: Populate `RenderTargetBinding` with color target resource, RTV, format, states, viewport, scissor, and `transitionMode` [S1, unofficial].
8.  **Render**: Call `renderer.Render(commandList, targetBinding, sceneHandle, input, frameContext, &result)` [S1, unofficial].
    - If `result.submission.uploadSyncPoint` is valid, wait on the direct queue for that sync point before executing the command list [S1, unofficial].
9.  **Submit**: Close the command list, execute it on the direct queue via `ExecuteCommandLists`, and signal the fence with `submissionFenceValue` [S1, unofficial].
10. **Update Fence**: On subsequent frames, update `completedFenceValue` with the new fence completion progress [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
context = D3D12Context()
context.Initialize(device, directQueue, directFence)

renderer = Renderer()
renderer.Initialize(context)

# Scene Loading
loaded_scene = LoadSceneFromFile(scene_path)
if not loaded_scene.ok:
    return loaded_scene.status

scene_handle = renderer.CreateUploadedScene(loaded_scene.value)

# Frame Loop
def render_frame(command_list, color_target, color_rtv, color_format, 
                 state_before, state_after, viewport, scissor, input, 
                 submission_fence_value):
    
    # 1. Frame Context
    frame_context = RenderFrameContext()
    frame_context.fence = directFence
    frame_context.completedFenceValue = directFence.GetCompletedValue()
    frame_context.submissionFenceValue = submission_fence_value
    frame_context.frameIndex = input.frameIndex
    
    # 2. Prepare
    preparation_result = RenderPreparationResult()
    status = renderer.PrepareSceneForRender(scene_handle, input, frame_context, preparation_result)
    if not status.ok:
        return status

    # 3. Target Binding
    target = RenderTargetBinding()
    target.colorTarget = color_target
    target.colorRtv = color_rtv
    target.colorFormat = color_format
    target.colorStateBefore = state_before
    target.colorStateAfter = state_after
    target.transitionMode = ResourceTransitionMode.LibraryManaged
    target.viewport = viewport
    target.scissor = scissor

    # 4. Render
    result = RenderResult()
    status = renderer.Render(command_list, target, scene_handle, input, frame_context, result)
    
    # Handle error with pending submission
    if not status.ok and result.submission.submissionRequired:
        # Command list work may already be recorded; must submit/signal or cleanup
        pass

    # 5. Sync Uploads (if needed)
    if result.submission.uploadSyncPoint.IsValid():
        hr = directQueue.Wait(result.submission.uploadSyncPoint.fence, 
                              result.submission.uploadSyncPoint.value)
        if FAILED(hr):
            renderer.NotifyDeviceLost()
            return Status.Error("Upload sync wait failed")

    # 6. Submit
    hr = command_list.Close()
    if FAILED(hr):
        renderer.NotifyDeviceLost()
        return Status.Error("Command list close failed")

    command_lists = [command_list]
    directQueue.ExecuteCommandLists(1, command_lists)
    
    hr = directQueue.Signal(directFence, frame_context.submissionFenceValue)
    if FAILED(hr):
        renderer.NotifyDeviceLost()
        return Status.Error("Fence signal failed")

    return status

# Shutdown
renderer.DestroyUploadedScene(scene_handle)
renderer.Shutdown()
context.Shutdown()
```

### Parameters and Typical Values
- `residencyBudgetGaussians`: 16,777,216 (16M) [S1, unofficial]
- `uploadBudgetGaussians`: 650,000 [S1, unofficial]
- `warmUploadBudgetGaussians`: 1,200,000 [S1, unofficial]
- `maxUploadsPerFrame`: 16 [S1, unofficial]
- `chunkTargetSize`: 65,536 [S1, unofficial]
- `lod0ScreenRadius`: 72.0f [S1, unofficial]
- `lod1ScreenRadius`: 24.0f [S1, unofficial]
- `cullScreenRadius`: 0.35f [S1, unofficial]
- `residencyCacheFrames`: 120 [S1, unofficial]

### Edge Cases and Pitfalls
- **Fence Values**: `submissionFenceValue` must be nonzero, unique per frame, strictly greater than every previously signaled value on that fence, and strictly greater than `completedFenceValue` [S1, unofficial].
- **Format Mismatch**: `colorRtv` must describe `colorTarget`, and `colorFormat` must match the RTV's single-sampled format; unsupported formats fail pipeline creation [S1, unofficial].
- **Error Submission**: If `Render` returns an error with `submissionRequired == true`, command-list work may already be recorded. The host must submit/signal it or enter device-lost cleanup [S1, unofficial].
- **Mutual Exclusion**: Rendering and scene mutation are mutually exclusive for the same uploaded scene. `BeginSceneMutation` waits for active render encoding to finish [S1, unofficial].
- **Resource Ownership**: `D3D12Context` stores non-owning pointers. The host must keep device, queues, and fences alive until `Shutdown()` completes [S1, unofficial].

## Operating characteristics
- **Performance**: Benchmarked on NVIDIA GeForce RTX 4070 SUPER, Windows 11. DirectXSplat achieves up to 2.4x faster FPS than `gsplat` in resident-scene benchmarks (e.g., 382.51 FPS vs 165.82 FPS on 'bicycle' dataset at 1237x822 resolution) [S1, unofficial].
- **Memory**: Memory-conscious design with persistent uploaded scenes and configurable residency/upload budgets [S1, unofficial].
- **Strengths**: High performance via D3D12 rasterization; native embeddability; explicit resource management; support for multiple scene formats (PLY, SPZ, .splat, SOG) [S1, unofficial].
- **Limitations**: Tied to Direct3D 12; requires host-managed synchronization and resource state transitions; not suitable for Vulkan/Metal without porting [S1, unofficial].
- **When to use**: When integrating 3D Gaussian Splatting into a D3D12 application requiring high performance and fine-grained control over the rendering pipeline [S1, unofficial].
- **When not to use**: For cross-platform rendering requiring Vulkan/Metal support, or for simple prototyping where convenience wrappers (`Draw`) are sufficient [S1, unofficial].

## Background
DirectXSplat is a C++/Direct3D 12 library developed to render scenes captured via 3D Gaussian Splatting techniques [S1, unofficial]. It is designed to be a fast, GPU-accelerated alternative to existing solutions like `gsplat`, focusing on native D3D12 integration and host-owned resource management [S1, unofficial]. The library supports various scene formats including PLY, compressed PLY, SPZ, .splat, and SOG, along with hierarchical LOD metadata [S1, unofficial].

## Sources
[S1] pbkx/DirectXSplat, https://github.com/pbkx/DirectXSplat, fetched 2026-10-27 (unofficial)
