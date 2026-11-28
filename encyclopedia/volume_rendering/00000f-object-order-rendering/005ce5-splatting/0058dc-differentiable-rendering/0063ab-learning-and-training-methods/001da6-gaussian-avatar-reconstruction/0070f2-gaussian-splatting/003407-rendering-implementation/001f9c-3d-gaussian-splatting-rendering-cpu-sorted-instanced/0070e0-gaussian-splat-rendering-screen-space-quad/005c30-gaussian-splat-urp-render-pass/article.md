# Gaussian Splat URP Render Pass

Also called: GSRenderPass, GaussianSplatURPFeature

## Overview
A Unity Universal Render Pipeline (URP) `ScriptableRendererFeature` that renders pre-sorted 3D Gaussian splats into a temporary render target and composites the result onto the camera color buffer. It differs from raw shader implementations by integrating into the URP render graph, handling XR multipass separation, and managing the intermediate render target lifecycle automatically.

## How it works
The method operates as a URP ScriptableRendererFeature. It intercepts the rendering pipeline to inject a custom render pass that executes after culling but before transparent rendering.

### Inputs and Outputs
- **Input**: Camera data (`CameraData`), XR state, and a `GaussianSplatSystem` instance containing splat data and sorting logic.
- **Output**: The composited Gaussian splats appear in the final camera color target.

### Data Structures
The feature maintains a dictionary of render passes keyed by eye ID to support XR multipass rendering.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```csharp
// Internal state
Dictionary<int, GSRenderPass> m_PassPerEye;
CommandBuffer m_Cmb; // Command buffer for the current frame
RenderTexture m_RenderTarget; // Temporary target for splat accumulation
Material matComposite; // Material for blitting splats to screen
```

The temporary render target `m_RenderTarget` is configured with:
- Format: `R16G16B16A16_SFloat` (16-bit float per channel)
- Dimensions: `Tex2D`
- Depth buffer: None (`depthBufferBits=0`)
- MSAA: Disabled (`msaaSamples=1`)
- Name: `"_GaussianSplatRT"`

### Numbered Steps

1. **Initialization**: In `OnCameraSetup`, allocate `m_RenderTarget` if needed using the descriptor above. Set it as a global texture. Configure the render pass to render to `m_RenderTarget` and read from the camera depth buffer. Clear the target with `Color(0,0,0,0)`.

2. **Pass Creation**: In `GetOrCreatePass`, determine the `eyeId`. If `eyeId` is not in `m_PassPerEye`, create a new `GSRenderPass` with the specified `renderPassEvent` and a named `CommandBuffer` (`'RenderGaussianSplats_{eyeId}'`). Store and return it.

3. **Event Hooking**:
   - In `OnCameraPreCull`: If the camera is a Preview, return. Otherwise, call `GetOrCreatePass` for `GetEyeId(cameraData)`.
   - In `AddRenderPasses`: If Preview, return. Get `eyeId`, get/create the pass, set `pass.m_Renderer = renderer`, and enqueue the pass.

4. **Execution**: In the pass's `Execute` method:
   - If `m_Cmb` is null, return.
   - Get the system and camera.
   - Call `system.GatherSplatsForCamera(cam)`. If it returns false, return.
   - Clear `m_Cmb`.
   - Call `system.SortAndRenderSplats(cam, m_Cmb)` to populate the command buffer and retrieve `matComposite`. If `matComposite` is null, return.
   - Begin profiling sample `s_ProfCompose`.
   - Blit `m_RenderTarget` to `cameraColorTargetHandle` using `matComposite` pass 0.
   - End profiling sample.
   - Execute `m_Cmb`.

5. **Eye ID Logic**: `GetEyeId` returns `cameraData.xr.multipassId` if XR is enabled, otherwise `cameraData.camera.GetInstanceID()`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class GaussianSplatURPFeature(ScriptableRendererFeature):
    def __init__(self):
        self.m_PassPerEye = {}
        self.m_RenderTarget = None
        self.matComposite = None
        self.renderPassEvent = RenderPassEvent.BeforeRenderingTransparents

    def OnCameraSetup(self, cameraData, renderTargetDescriptor):
        # Allocate temporary render target
        desc = RenderTextureDescriptor()
        desc.width = renderTargetDescriptor.width
        desc.height = renderTargetDescriptor.height
        desc.graphicsFormat = GraphicsFormat.R16G16B16A16_SFloat
        desc.dimension = TextureDimension.Tex2D
        desc.depthBufferBits = 0
        desc.msaaSamples = 1
        desc.vrUsage = VRUsage.None
        
        if self.m_RenderTarget is None or not self.m_RenderTarget.IsCreated():
            self.m_RenderTarget = RenderTexture.GetTemporary(desc)
            self.m_RenderTarget.name = "_GaussianSplatRT"
            Shader.SetGlobalTexture("_GaussianSplatRT", self.m_RenderTarget)
            
        # Configure clear
        self.m_RenderTarget.ClearColor = Color(0, 0, 0, 0)

    def GetEyeId(self, cameraData):
        if cameraData.xr.enabled:
            return cameraData.xr.multipassId
        else:
            return cameraData.camera.GetInstanceID()

    def GetOrCreatePass(self, eyeId):
        if eyeId not in self.m_PassPerEye:
            pass_instance = GSRenderPass()
            pass_instance.renderPassEvent = self.renderPassEvent
            pass_instance.commandBuffer = CommandBuffer()
            pass_instance.commandBuffer.name = f"RenderGaussianSplats_{eyeId}"
            self.m_PassPerEye[eyeId] = pass_instance
        return self.m_PassPerEye[eyeId]

    def OnCameraPreCull(self, cameraData):
        if cameraData.cameraType == CameraType.Preview:
            return
        eye_id = self.GetEyeId(cameraData)
        self.GetOrCreatePass(eye_id)

    def AddRenderPasses(self, renderer, cameraData):
        if cameraData.cameraType == CameraType.Preview:
            return
        eye_id = self.GetEyeId(cameraData)
        pass_instance = self.GetOrCreatePass(eye_id)
        pass_instance.renderer = renderer
        renderer.EnqueuePass(pass_instance)

    class GSRenderPass(ScriptableRenderPass):
        def __init__(self):
            self.m_Cmb = CommandBuffer()
            self.matComposite = None
            
        def Execute(self, context, refData):
            if self.m_Cmb is None:
                return
                
            system = GetGaussianSplatSystem() # [G] Assumed global or passed via feature
            camera = refData.camera
            
            if not system.GatherSplatsForCamera(camera):
                return
                
            self.m_Cmb.Clear()
            self.matComposite = system.SortAndRenderSplats(camera, self.m_Cmb)
            
            if self.matComposite is None:
                return
                
            Profiler.BeginSample("ComposeGaussianSplats")
            Blit(self.m_RenderTarget, refData.cameraColorTargetHandle, self.matComposite, 0)
            Profiler.EndSample()
            
            context.ExecuteCommandBuffer(self.m_Cmb)
```

### Parameters
- `renderPassEvent`: `RenderPassEvent.BeforeRenderingTransparents` [S1, unofficial].

### Edge Cases and Pitfalls
- **Preview Cameras**: The feature explicitly skips execution for `CameraType.Preview` to avoid overhead in the editor [S1, unofficial].
- **XR Multipass**: The use of `multipassId` ensures separate render passes for each eye in VR, preventing cross-eye contamination [S1, unofficial].
- **Precision**: The use of `R16G16B16A16_SFloat` provides sufficient precision for HDR accumulation, preventing banding in bright areas [S1, unofficial].

## Subtopics

### [WebGL Gaussian Splatting](2084-webgl-gaussian-splatting.md)
Renders splats in a web browser context using WebGL, lacking URP integration.

### [Debug Splat Point Rendering](2101-debug-splat-point-rendering.md)
Visualizes splat centers as points for debugging, rather than full splat rendering.

### [Gaussian Splat Fragment Shader](3258-gaussian-splat-fragment-shader.md)
The low-level fragment shader logic for blending, used within the URP pass.

### [Gaussian Splat Shader Property Configuration](3596-gaussian-splat-shader-property-configuration.md)
Manages shader properties for the splatting material.

### [Animated 3D Gaussian Splatting (Splatv rendering)](7758-animated-3d-gaussian-splatting-splatv-rendering.md)
Handles time-varying splat parameters, requiring additional animation updates in the render pass.

## Operating characteristics
- **Time Complexity**: Dominated by `SortAndRenderSplats`, which is $O(N \log N)$ for sorting and $O(N)$ for rendering, where $N$ is the number of splats.
- **Memory Complexity**: $O(N)$ for splat data storage and $O(W \times H)$ for the temporary render target.
- **Performance**: Depends heavily on the efficiency of `GatherSplatsForCamera` and the sorting algorithm. The use of a temporary render target adds a memory bandwidth cost for the blit operation.
- **Strengths**: Seamless integration with URP, automatic XR support, and clean separation of concerns.
- **Limitations**: Requires Unity URP; not portable to other engines or raw OpenGL/Vulkan without adaptation.

## Background
This method is part of the `worldlabs_unity` repository, an unofficial implementation of 3D Gaussian Splatting for Unity. It adapts the CPU-sorted, instanced rendering approach to the Unity URP framework.

## Sources
[S1] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
