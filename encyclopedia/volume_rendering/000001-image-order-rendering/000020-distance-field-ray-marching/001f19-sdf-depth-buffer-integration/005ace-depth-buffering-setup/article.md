# Depth Buffering Setup

Also called: Depth State Setup, Depth Write Configuration

## Overview
This method configures the graphics pipeline depth state so that ray-marched surfaces in a Signed Distance Field (SDF) volume rendering pass correctly participate in the depth buffer [S1, unofficial]. By writing the correct depth value to `SV_Depth`, the rendered volume occludes geometry behind it and is itself occluded by geometry in front, solving incorrect depth ordering in hybrid pipelines [G]. It differs from [Deferred SDF Rendering](13554-deferred-sdf-rendering.md) by focusing specifically on immediate depth state configuration rather than deferring lighting computation [G].

## How it works
**Inputs:** Current render target and depth stencil view, depth comparison function, depth write flag [G].
**Outputs:** Configured depth stencil state [G].
**Data Structures:** None created by this step; relies on an existing depth buffer populated by prior passes [G].
**Steps:**
1. Set the depth comparison function to `Less` [S1, unofficial].
2. Enable the depth write mask [S1, unofficial].
3. Apply the configured depth stencil state to the current render pass or pipeline descriptor [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_depth_buffering(compare_func: str = "less", write_enabled: bool = True):
    class DepthStencilState:
        def __init__(self, compare_func, write_enabled):
            self.compare_func = compare_func
            self.write_enabled = write_enabled
            self.stencil_front = {"fail": "keep", "depth_fail": "keep", "pass": "keep"}
            self.stencil_back = {"fail": "keep", "depth_fail": "keep", "pass": "keep"}
            self.mask = 0xFF
            self.write_mask = 0xFF if write_enabled else 0x00

    depth_state = DepthStencilState(compare_func=compare_func, write_enabled=write_enabled)
    pipeline_descriptor = {"depth_stencil": depth_state}
    return pipeline_descriptor
```
**Parameters:** `compare_func`: `Less` [S1, unofficial]. `write_enabled`: `True` [S1, unofficial].
**Edge Cases and Pitfalls:**
- **Ordering:** Opaque geometry must be rendered before the volume pass to ensure correct occlusion [G].
- **Clamping:** Depth values written by the ray marcher must be within the valid API range (e.g., `[0, 1]` for OpenGL/Vulkan, `[-1, 1]` for DirectX) to avoid undefined behavior [G].
- **Precision:** Depth buffer bit depth affects occlusion accuracy for large scenes or thin geometry; 24-bit or 32-bit buffers are recommended for high precision [G].

## Subtopics

### [Depth Buffer Integration for Occlusion](16810-depth-buffer-integration-for-occlusion.md)
This variant reuses the depth buffer from a preliminary object rendering pass to discard volume fragments lying behind opaque geometry. Unlike Extended Depth Peeling, which uses wider area occluders for higher accuracy at greater computational cost, this method treats a single back-projected pixel as the occluder, offering a lighter-weight alternative for deep layers where visibility gains from complex culling diminish.

## Operating characteristics
- **Time Complexity:** O(1) for state setup; per-fragment occlusion checks add negligible overhead to ray marching [G].
- **Memory Complexity:** O(1) for state setup; no additional memory allocated [G].
- **Performance:** Culls volume fragments occluded by opaque geometry, reducing ray marching steps [G].
- **Strengths:** Simple to implement, effective for immediate occlusion culling, integrates with standard pipelines [G].
- **Limitations:** Requires correct pass ordering; does not handle transparent geometry or complex lighting directly [G].

## Background
Not given in the sources.

## Sources
[S1] statusfailed/wgpu-voxel-terrain, https://github.com/statusfailed/wgpu-voxel-terrain, fetched 2026-10-27 (unofficial)
