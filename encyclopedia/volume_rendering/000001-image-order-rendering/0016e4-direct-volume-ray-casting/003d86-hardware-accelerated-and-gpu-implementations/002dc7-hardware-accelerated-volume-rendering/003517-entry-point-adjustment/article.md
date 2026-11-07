# Entry Point Adjustment

Also called: Flowing through volume, Inside-volume rendering, Raycasting Volume Rendering with Entry Point Adjustment

## Overview
Entry Point Adjustment is a hardware-accelerated volume rendering technique that enables the camera to move inside the volume's bounding box without the view disappearing or becoming invalid. It solves the problem where standard ray casting fails when the camera position exceeds the front face of the volume, by dynamically shifting the ray's entry point forward along the ray direction based on a scale coefficient. It differs from standard hardware-accelerated rendering [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md) by decoupling the visual entry point from the geometric front face, allowing for a "flowing through" effect.

## How it works
The method extends the standard two-pass hardware-accelerated approach. It requires an additional render pass to capture exit points and a modified fragment shader to handle dynamic entry point adjustment.

### Inputs and Outputs
*   **Inputs:**
    *   `VolumeTex`: 3D texture containing scalar volume data.
    *   `TransferFunc`: 1D texture mapping scalar values to RGBA color/opacity.
    *   `ExitPoints`: 2D texture storing the 3D exit coordinates of rays passing through the back face of the volume bounding box.
    *   `ScreenSize`: Uniform `vec2` representing the resolution of the viewport.
    *   `StepSize`: Uniform `float` controlling the distance between samples.
    *   `AlphaReduce`: Uniform `float` multiplier to reduce opacity.
    *   `scaleCoeff`: Uniform `float` representing the scaling factor of the volume/camera, used to determine if the camera is inside.
    *   `EntryPoint`: Varying `vec3` passed from the vertex shader, representing the geometric entry point (front face intersection).
*   **Outputs:**
    *   `FragColor`: `vec4` containing the accumulated color and opacity for the pixel.

### Data Structures
*   **ExitPoints Texture:** A 2D texture (`sampler2D`) where each texel at normalized screen coordinates $(u, v)$ stores the 3D world position $(x, y, z)$ of the ray's exit point from the volume's back face. The coordinates are stored in the RGB channels.

### Steps
1.  **Back-Face Rendering:** Render the back face of the volume's bounding box geometry into a framebuffer object (FBO) attached to a 2D texture named `ExitPoints`. The vertex shader outputs the world position of the back-face vertices. This captures the exit point for every ray passing through the volume.
2.  **Front-Face Rendering:** Render the front face of the volume's bounding box to the screen. The vertex shader outputs the geometric entry point (`EntryPoint`) and the screen coordinates.
3.  **Fragment Shader Execution:** For each pixel:
    a. Retrieve the exit point from `ExitPoints` using the normalized screen coordinates (`gl_FragCoord.st / ScreenSize`).
    b. Check if the ray intersects the volume. If `EntryPoint` equals `exitPoint` (indicating no intersection or camera outside), discard the fragment.
    c. Calculate the ray direction: $\mathbf{d} = \text{normalize}(\text{exitPoint} - \text{EntryPoint})$.
    d. **Entry Point Adjustment:** If `scaleCoeff` $\ge 2.7$, calculate a temporary shift factor:
       $$ \text{tmp} = \min((\text{scaleCoeff} - 2.7) \times 0.1, 1.0) $$
       Adjust the entry point:
       $$ \text{adjustedEntryPoint} = \text{EntryPoint} + \text{tmp} \times (\text{exitPoint} - \text{EntryPoint}) $$
       Otherwise, use the original `EntryPoint`.
    e. Initialize the current position at `adjustedEntryPoint`. Initialize accumulation variables: `colorSum = (0,0,0,0)`, `color = (0,0,0,0)`, `value = (0,0,0,0)`.
    f. Calculate the step vector: $\mathbf{s} = \mathbf{d} \times \text{StepSize}$.
    g. Calculate the total ray length: $L = \|\text{exitPoint} - \text{EntryPoint}\|$.
    h. **Ray Marching Loop:** Iterate up to 16,000 times (or until termination conditions are met):
       i. Sample the volume: $V = \text{texture}(\text{VolumeTex}, \text{currentPosition.xyz})$.
       ii. Sample the transfer function: $C = \text{texture}(\text{TransferFunc}, V.a)$.
       iii. Reduce alpha: $C.a = C.a \times \text{AlphaReduce}$.
       iv. Front-to-back compositing:
           $$ C_{\text{contrib}} = C.rgb \times C.a $$
           $$ \text{colorSum} = \text{colorSum} + (1.0 - \text{colorSum.a}) \times C_{\text{contrib}} $$
           $$ \text{colorSum.a} = \text{colorSum.a} + (1.0 - \text{colorSum.a}) \times C.a $$
       v. Accumulate length: $L_{\text{sum}} += \|\mathbf{s}\|$.
       vi. **Early Ray Termination:** If $\text{colorSum.a} \ge 0.95$, break.
       vii. Advance position: $\text{currentPosition.xyz} += \mathbf{s}$.
       viii. **Boundary Check:** If $L_{\text{sum}} \ge L$, break.
    i. Output `colorSum` as `FragColor`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic
def fragment_shader(EntryPoint: vec3, ExitPointCoord: vec4, 
                    ExitPoints: sampler2D, VolumeTex: sampler3D, 
                    TransferFunc: sampler1D, StepSize: float, 
                    AlphaReduce: float, ScreenSize: vec2, 
                    scaleCoeff: float) -> vec4:
    
    # 1. Retrieve exit point from texture
    normalized_coords = gl_FragCoord.st / ScreenSize
    exitPoint = texture(ExitPoints, normalized_coords).xyz
    
    # 2. Check for intersection
    if EntryPoint == exitPoint:
        discard_fragment()
        
    # 3. Calculate ray direction
    rayDirection = normalize(exitPoint - EntryPoint)
    
    # 4. Entry Point Adjustment
    if scaleCoeff >= 2.7:
        tmp = min((scaleCoeff - 2.7) * 0.1, 1.0)
        adjustedEntryPoint = EntryPoint + tmp * (exitPoint - EntryPoint)
    else:
        adjustedEntryPoint = EntryPoint
        
    # 5. Initialize variables
    currentPosition = adjustedEntryPoint
    colorSum = vec4(0.0, 0.0, 0.0, 0.0)
    Step = rayDirection * StepSize
    stepLength = length(Step)
    Length = length(exitPoint - EntryPoint)
    LengthSum = 0.0
    
    # 6. Ray Marching Loop
    for i in range(16000):
        # Sample volume and transfer function
        value = texture(VolumeTex, currentPosition).a
        color = texture(TransferFunc, value)
        
        # Reduce alpha
        color.a *= AlphaReduce
        
        # Front-to-back blending
        color_contrib = color.rgb * color.a
        colorSum = colorSum + (1.0 - colorSum.a) * color_contrib
        colorSum.a = colorSum.a + (1.0 - colorSum.a) * color.a
        
        # Accumulate length
        LengthSum += stepLength
        
        # Early Ray Termination
        if colorSum.a >= 0.95:
            break
            
        # Advance position
        currentPosition += Step
        
        # Boundary check
        if LengthSum >= Length:
            break
            
    return colorSum
```

### Parameters
*   `StepSize`: Controls sampling density. Smaller values increase quality but decrease performance.
*   `AlphaReduce`: Reduces opacity to make the volume more transparent, useful for internal visualization.
*   `scaleCoeff`: Threshold for entering the "inside volume" mode. Values $\ge 2.7$ trigger the entry point shift.

### Edge Cases and Pitfalls
*   **Precision:** Floating-point precision errors may cause `EntryPoint == exitPoint` checks to fail incorrectly if the camera is exactly on the boundary. Use a small epsilon for comparison if necessary [G].
*   **Degenerate Input:** If the volume bounding box is not axis-aligned or the camera projection is non-standard, the simple back-face rasterization may not capture correct exit points. This method assumes standard perspective or orthographic projection where back-face rasterization correctly maps to exit coordinates [S1, unofficial].
*   **Performance:** The loop limit of 16,000 is arbitrary and may be excessive for small volumes. It should be tuned based on the volume size and `StepSize` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray, limited by the loop count (16,000) and early termination.
*   **Memory Complexity:** $O(W \times H)$ for the `ExitPoints` texture, where $W$ and $H$ are the screen dimensions.
*   **Strengths:** Enables seamless navigation inside the volume without visual discontinuities. Simple to implement on standard GPU hardware.
*   **Limitations:** The entry point adjustment is heuristic-based (`scaleCoeff` threshold) and may not produce physically accurate results for all camera positions. It does not handle occlusion from internal structures as effectively as more advanced methods like ray marching with spatial acceleration structures.
*   **When to use:** When visualizing internal structures by "flying through" the volume is required, and hardware acceleration is needed.
*   **When not to use:** When high accuracy of internal ray origins is critical, or when the volume has complex internal occlusion that requires more sophisticated traversal.

## Background
This technique originates from practical implementations of hardware-accelerated volume rendering where users needed to navigate inside the volume. It was discussed in online forums as a solution to the "disappearing view" problem when the camera moves past the front face of the bounding box [S1, unofficial]. It is often contrasted with Plane Clipping, which is another suggested method for handling visibility through the volume [S1, unofficial].

## Sources
[S1] Volume rendering using ray-casting- flowing through the volume, https://stackoverflow.com/questions/36436620/volume-rendering-using-ray-casting-flowing-through-the-volume, fetched 2026-10-26 (unofficial)
