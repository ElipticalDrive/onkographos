# Frustum Raycasting and Blending

Also called: FragmentProgram, OnRenderImage

## Overview
Frustum Raycasting and Blending is a screen-space post-processing technique that casts rays from the camera through each pixel to intersect a cloud volume bounding box, blending the resulting cloud color with the original scene color [S1, unofficial]. It addresses the artifacts of standard alpha blending (incorrect transparency order) and alpha testing (hard edges) by performing a screen-space depth-difference check during the final compositing pass [S1, unofficial]. It differs from [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md) by explicitly using ray-box intersection logic in the fragment shader rather than pre-composited textures, and from [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md) by focusing on volumetric occlusion rather than luminance scaling.

## How it works
**Inputs and Outputs:**
*   **Input:** Scene texture (`_MainTex`), UV coordinates (`input.uv`), camera world position (`_CameraWorldPosition`), cloud volume bounds (`_CloudVolumeMinBound`, `_CloudVolumeMaxBound`), and frustum corners (`_FrustumCornerTR`, `_FrustumCornerBR`).
*   **Output:** Blended color (`finalColor`).

**Data Structures:**
*   `Vector3`: Used for positions, directions, and bounds.
*   `Vector2`: Used for UV coordinates.
*   `float`: Used for distances and color components.

**Steps:**
1.  **Sample Scene Color:** Retrieve the original scene color from `_MainTex` using the fragment's UV coordinates.
    $$ C_{scene} = \text{tex2D}(\text{_MainTex}, \text{input.uv}) $$
2.  **Initialize Ray:** Set the ray origin to the camera's world position.
    $$ O = \text{_CameraWorldPosition} $$
3.  **Compute Ray Direction:** Calculate the direction vector from the camera to the current pixel. Normalize this vector.
    $$ D = \text{normalize}(\text{direction}) $$
    [G] The direction is typically derived from the screen-space position transformed into world space, or by using the pre-computed frustum corners if the shader is applied as a full-screen quad.
4.  **Ray-Box Intersection:** Calculate the intersection of the ray $(O, D)$ with the axis-aligned bounding box defined by `_CloudVolumeMinBound` and `_CloudVolumeMaxBound`. This returns entry distance $t_{entry}$ and exit distance $t_{exit}$.
    $$ (t_{entry}, t_{exit}) = \text{RayBoxIntersection}(O, D, \text{_CloudVolumeMinBound}, \text{_CloudVolumeMaxBound}) $$
5.  **March Density:** If $t_{entry} < t_{exit}$ (indicating an intersection), march along the ray segment $[t_{entry}, t_{exit}]$ to sample the cloud density and compute the cloud color.
    $$ C_{cloud} = \text{MarchDensity}(t_{entry}, t_{exit}, O, D, \text{_CloudVolumeMinBound}, \text{_CloudVolumeMaxBound}) $$
    [G] `MarchDensity` typically involves stepping through the volume, sampling a 3D noise texture (e.g., Perlin noise) scaled by a frequency, accumulating density, and applying a scatter factor to determine opacity and color.
6.  **Blend:** If an intersection occurred, blend the cloud color with the scene color using the cloud's alpha.
    $$ C_{final} = \text{lerp}(C_{scene}, C_{cloud}, C_{cloud}.a) $$
    If no intersection occurred, return the original scene color.
    $$ C_{final} = C_{scene} $$

**C# Script Setup (OnRenderImage):**
1.  In the `OnRenderImage` method, calculate the frustum corners at the far clip plane using `Camera.CalculateFrustumCorners`.
2.  Transform these corners from camera space to world space using `camera.transform.TransformDirection`.
3.  Pass the transformed corners to the shader as `_FrustumCornerTR` (Top-Right) and `_FrustumCornerBR` (Bottom-Right).
4.  Set the cloud volume bounds from the scene object's `Renderer.bounds`.
    ```csharp
    void OnRenderImage(RenderTexture source, RenderTexture destination)
    {
        if (CloudShaderMaterial != null && CloudVolumeBounds != null)
        {
            Bounds bounds = CloudVolumeBounds.GetComponent<Renderer>().bounds;
            CloudShaderMaterial.SetVector("_CloudVolumeMinBound", bounds.min);
            CloudShaderMaterial.SetVector("_CloudVolumeMaxBound", bounds.max);
            CloudShaderMaterial.SetFloat("_DensityThreshold", densityThreshold);
            CloudShaderMaterial.SetFloat("_ScatterFactor", scatterFactor);
            
            // Calculate and set frustum corners
            Camera cam = GetComponent<Camera>();
            Vector3[] corners = new Vector3[4];
            cam.CalculateFrustumCorners(new Rect(0, 0, 1, 1), cam.farClipPlane, Camera.MonoOrStereoscopicEye.Mono, corners);
            // Transform to world space
            for (int i = 0; i < 4; i++) {
                corners[i] = cam.transform.TransformDirection(corners[i]);
            }
            CloudShaderMaterial.SetVector("_FrustumCornerTR", corners[2]); // Top-Right
            CloudShaderMaterial.SetVector("_FrustumCornerBR", corners[3]); // Bottom-Right
        }
        Graphics.Blit(source, destination, CloudShaderMaterial);
    }
    ```

**Parameters:**
*   `_DensityThreshold`: Threshold for density accumulation to determine opacity.
*   `_ScatterFactor`: Factor controlling light scattering within the volume.
*   `windSpeed`: Speed at which the wind offset increases, used to animate noise sampling coordinates.

**Edge Cases and Pitfalls:**
*   **Precision:** Ray-box intersection calculations can suffer from floating-point precision errors near the box boundaries.
*   **Degenerate Input:** If the camera is inside the cloud volume, $t_{entry}$ may be negative, requiring clamping to 0.
*   **Performance:** Ray marching is computationally expensive; the number of steps in `MarchDensity` must be limited.

## Subtopics

### [Cloud-Scene Occlusion Blending](20794-cloud-scene-occlusion-blending.md)
A technique that uses depth information from opaque objects and the cloud layer to determine visibility, preventing clouds from appearing in front of opaque objects. It is chosen when correct occlusion is critical, unlike standard alpha blending.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps in the ray march.
*   **Memory Complexity:** $O(1)$ per pixel, excluding the texture memory for the noise volume.
*   **Performance:** Dependent on the resolution and the number of marching steps. High resolutions or large volumes can cause significant performance drops.
*   **Strengths:** Provides accurate occlusion and blending without pre-computation; works with dynamic cloud volumes.
*   **Limitations:** Computationally intensive; requires careful tuning of parameters to avoid artifacts.
*   **When to Use:** When high-quality volumetric clouds are needed and performance allows for per-pixel ray marching.
*   **When Not to Use:** When performance is critical or the cloud volume is static and can be pre-composited.

## Background
This method is part of a volumetric cloud renderer implementation, often associated with Unity shader programming. It builds on the principles of direct volume ray casting and screen-space post-processing. The use of `OnRenderImage` indicates its integration into a Unity-based rendering pipeline. The noise generation often relies on Perlin noise, as referenced in the source code comments.

## Sources
[S1] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
