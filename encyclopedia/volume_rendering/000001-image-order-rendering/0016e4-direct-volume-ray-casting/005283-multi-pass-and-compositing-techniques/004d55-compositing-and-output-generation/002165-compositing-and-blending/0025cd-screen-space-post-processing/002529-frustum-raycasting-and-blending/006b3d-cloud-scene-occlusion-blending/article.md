# Cloud-Scene Occlusion Blending

Also called: AlphaBlendWithDepthDiff, Mix Cloud With Scenes

## Overview
Cloud-Scene Occlusion Blending is a post-processing technique that blends rendered clouds with the scene by using depth information from opaque objects and the cloud layer to determine visibility and occlusion, preventing clouds from appearing in front of opaque objects incorrectly [S1, unofficial]. It addresses the artifacts of standard alpha blending (incorrect transparency order) and alpha testing (hard edges) by performing a screen-space depth-difference check during the final compositing pass [S1, unofficial].

## How it works
The method operates in three phases: buffer preparation, cloud rendering, and screen-space compositing.

### Inputs and Outputs
**Inputs:**
*   Scene color and depth from opaque objects (rendered before clouds).
*   Cloud color and depth (rendered after opaque objects).
*   Parameters: `occludeeVisibility` (default 0.35f), `occludeeVisibleRange` (default 0.6f) [S1, unofficial].

**Outputs:**
*   Final screen image with correctly occluded clouds.

### Data Structures
Six `RenderTexture` objects are required. The specific formats are:
1.  `ZBuffer`: Depth, 24-bit.
2.  `ColorBuffer`: RGB111110Float.
3.  `CT_opaque`: RGB111110Float (stores opaque scene color).
4.  `CT_cloud`: RGB111110Float (stores cloud color).
5.  `DT_opaque`: RHalf (stores opaque scene depth).
6.  `DT_cloud`: RHalf (stores cloud depth) [S1, unofficial].

### Steps
1.  **Buffer Allocation**: Create the six `RenderTexture` instances with the formats specified above [S1, unofficial].
2.  **Pre-Alpha Capture**: Add a `CommandBuffer` (`CB_BeforeAlpha`) at `CameraEvent.BeforeForwardAlpha`. This buffer blits the current `ColorBuffer` to `CT_opaque` and `ZBuffer` to `DT_opaque` [S1, unofficial].
3.  **Cloud Rendering Setup**: In `OnPreRender`, set the camera's target buffers to `ColorBuffer.colorBuffer` and `ZBuffer.depthBuffer`. Render the cloud geometry. The cloud shader should write depth (ZWrite On) and perform alpha blending or testing as needed to populate `ColorBuffer` and `ZBuffer` with cloud data [S1, unofficial].
4.  **Post-Alpha Capture**: Add a `CommandBuffer` (`CB_AfterAlpha`) at `CameraEvent.AfterForwardAlpha`. This buffer blits the current `ColorBuffer` (now containing clouds) to `CT_cloud` and `ZBuffer` (now containing cloud depth) to `DT_cloud` [S1, unofficial].
5.  **Compositing**: In `OnPostRender`, set the shader properties for the mix material:
    *   `_OpaqueColorTex` = `CT_opaque`
    *   `_OpaqueDepthTex` = `DT_opaque`
    *   `_AllDepthTex` = `DT_cloud`
    *   `_OccludeeVisibility` = `occludeeVisibility`
    *   `_OccludeeVisibleRange` = `occludeeVisibleRange`
    *   Blit `CT_cloud` to the screen using the mix shader material [S1, unofficial].

### Shader Logic (Pseudocode)
The mix shader performs the blending. Let $D_{opaque}$ be the depth from `_OpaqueDepthTex`, $D_{cloud}$ be the depth from `_AllDepthTex`, and $C_{cloud}$ be the color from `_OpaqueColorTex` (scene) and $C_{cloud\_tex}$ be the color from `CT_cloud` (clouds). Note: The source uses `_OpaqueColorTex` for the scene color in the final blit context, but logically it blends the cloud color with the scene color. Based on the description "blends rendered clouds with the scene", the inputs are Scene Color ($C_{scene}$) and Cloud Color ($C_{cloud}$).

The visibility factor $\alpha_{blend}$ is determined by the depth difference.
Let $\Delta D = D_{cloud} - D_{opaque}$.
If $D_{cloud} < D_{opaque}$ (cloud is in front of opaque object), the cloud is fully visible ($\alpha_{blend} = 1$).
If $D_{cloud} > D_{opaque}$ (cloud is behind opaque object), the cloud is occluded. The visibility transitions based on `occludeeVisibleRange`.

$$
\alpha_{blend} = \begin{cases} 
1 & \text{if } D_{cloud} \le D_{opaque} \\
\text{clamp}\left(\frac{D_{opaque} - D_{cloud} + \text{occludeeVisibleRange}}{\text{occludeeVisibleRange}}, 0, 1\right) \times \text{occludeeVisibility} & \text{if } D_{cloud} > D_{opaque}
\end{cases}
$$

*Note: The exact formula for the transition is inferred from the parameter names `occludeeVisibility` (min alpha) and `occludeeVisibleRange` (transition width). Standard linear interpolation between 0 and 1 over the range is assumed, scaled by the minimum visibility.*

Final Color $C_{final}$:
$$
C_{final} = C_{scene} \times (1 - \alpha_{blend}) + C_{cloud} \times \alpha_{blend}
$$

[G] The depth values in `RHalf` textures must be read and compared correctly. In Unity, `_CameraDepthTexture` or custom depth textures often require decoding if stored in a specific format, but here `RHalf` implies direct float representation or standard depth buffer format. The comparison $D_{cloud} - D_{opaque}$ assumes linear depth or consistent projection space. If non-linear, depth decoding is required before subtraction.

### Pseudocode (C# Integration)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```csharp
// Setup buffers (called once or in OnPreRender)
RenderTexture ZBuffer = new RenderTexture(Screen.width, Screen.height, 24);
RenderTexture ColorBuffer = new RenderTexture(Screen.width, Screen.height, 0, RenderTextureFormat.ARGBHalf); // Using ARGBHalf for RGB111110Float equivalent in Unity
RenderTexture CT_opaque = new RenderTexture(Screen.width, Screen.height, 0, RenderTextureFormat.ARGBHalf);
RenderTexture CT_cloud = new RenderTexture(Screen.width, Screen.height, 0, RenderTextureFormat.ARGBHalf);
RenderTexture DT_opaque = new RenderTexture(Screen.width, Screen.height, 0, RenderTextureFormat.RHalf);
RenderTexture DT_cloud = new RenderTexture(Screen.width, Screen.height, 0, RenderTextureFormat.RHalf);

// Command Buffers
CommandBuffer CB_BeforeAlpha = new CommandBuffer { name = "CB_BeforeAlpha" };
CB_BeforeAlpha.Blit(BuiltinRenderTextureType.CurrentActive, CT_opaque); // Blit Color
CB_BeforeAlpha.Blit(BuiltinRenderTextureType.CurrentActive, DT_opaque); // Blit Depth (requires specific handling for depth-only)
// Note: Unity's Blit to depth texture requires specific setup or GrabPass-like behavior. 
// The source implies direct blitting. In practice, DT_opaque captures the depth buffer state.

CommandBuffer CB_AfterAlpha = new CommandBuffer { name = "CB_AfterAlpha" };
CB_AfterAlpha.Blit(BuiltinRenderTextureType.CurrentActive, CT_cloud);
CB_AfterAlpha.Blit(BuiltinRenderTextureType.CurrentActive, DT_cloud);

// Camera Events
camera.AddCommandBuffer(CameraEvent.BeforeForwardAlpha, CB_BeforeAlpha);
camera.AddCommandBuffer(CameraEvent.AfterForwardAlpha, CB_AfterAlpha);

// OnPreRender
void OnPreRender() {
    camera.SetTargetBuffers(ColorBuffer.colorBuffer, ZBuffer.depthBuffer);
}

// OnPostRender
void OnPostRender() {
    if (material != null) {
        material.SetTexture("_OpaqueColorTex", CT_opaque);
        material.SetTexture("_OpaqueDepthTex", DT_opaque);
        material.SetTexture("_AllDepthTex", DT_cloud);
        material.SetFloat("_OccludeeVisibility", occludeeVisibility);
        material.SetFloat("_OccludeeVisibleRange", occludeeVisibleRange);
        Graphics.Blit(CT_cloud, (RenderTexture)null, material);
    } else {
        Graphics.Blit(CT_cloud, (RenderTexture)null);
    }
}
```

### Parameters
*   `occludeeVisibility`: Minimum visibility of the occluded object (cloud). Range: $[0.01, 1]$. Default: $0.35$ [S1, unofficial].
*   `occludeeVisibleRange`: Range of depth difference for visibility transition. Range: $[0, 1]$. Default: $0.6$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Cloud-Cloud Occlusion**: The method primarily handles cloud-scene occlusion. If multiple cloud layers are rendered, ZWrite On may cause later clouds to fail ZTest against earlier clouds, leading to incorrect transparency at cloud boundaries ("showing background color") [S1, unofficial].
*   **Performance**: Creating multiple RenderTextures and CommandBuffers adds overhead. GrabPass-like operations (blitting entire frames) are expensive on mobile platforms [S1, unofficial].
*   **Depth Precision**: Using `RHalf` for depth may introduce precision issues at large distances or small depth differences, affecting the smoothness of the occlusion transition.

## Operating characteristics
*   **Time Complexity**: Linear with respect to screen resolution and number of render passes (3 passes: opaque, cloud, post-process).
*   **Memory Complexity**: Proportional to screen resolution $\times$ 6 textures.
*   **Strengths**: Soft edges at cloud-object boundaries; correct occlusion of clouds by opaque objects.
*   **Limitations**: Does not solve inter-cloud occlusion artifacts without CPU sorting (which breaks GPU instancing); expensive due to multiple frame buffer copies; sensitive to depth precision.
*   **When to use**: When soft cloud edges against opaque geometry are required and performance budget allows for post-processing.
*   **When not to use**: On mobile devices with limited memory/bandwidth; when strict cloud-to-cloud occlusion is needed without CPU sorting.

## Background
This technique is described in the context of Unity's Built-in Render Pipeline for rendering volume clouds. It is presented as an improvement over simple Alpha Blend (ZWrite Off) and Alpha Test (ZWrite On) methods to handle the visual artifacts of transparent cloud rendering against opaque scene geometry [S1, unofficial].

## Sources
[S1] ShuchangXu/VolumeCloud, https://github.com/ShuchangXu/VolumeCloud, fetched 2026-10-26 (unofficial)
