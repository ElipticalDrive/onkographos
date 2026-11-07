# Back-face rasterized volume rendering

Also called: Back-Face Entry Point Method, Backface Rendering for Ray Casting, Backface Culling for Ray Casting Entry Points

## Overview
Back-face rasterized volume rendering is a hardware-accelerated technique that determines ray entry and exit points for volume ray casting by rasterizing the back faces of the volume's bounding box into a texture [S1, S4, unofficial]. It solves the problem of efficiently culling pixels outside the volume and avoiding expensive ray-box intersection calculations in the final rendering pass [S2, S3, unofficial]. Unlike [Bounding Box Raymarching](2071-bounding-box-raymarching.md), which explicitly uses a two-pass approach to capture back-face coordinates, this method is the foundational implementation of that concept, often distinguishing itself by how it handles the transition between passes and the specific storage of boundary data [S1, S2, S3, unofficial]. It differs from [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md) by deriving ray bounds from geometry colors rather than storing ray data in a dedicated texture [S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume texture ($V$), transfer function texture ($TF$), camera parameters (eye position, projection/view matrices), bounding box geometry (unit cube or scaled cube).
*   **Output:** 2D color buffer with the rendered volume image.

**Data Structures:**
1.  **Bounding Box Geometry:** Defined by 8 vertices in object coordinates $[0,1]^3$ and 36 indices forming 12 triangles [S4, unofficial].
    *   Vertices: $\{(0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)\}$ [S4, unofficial].
2.  **Back-Face Texture ($g\_bfTexObj$):** A 2D texture (e.g., `GL_RGBA16F`) with resolution equal to the viewport ($W \times H$) [S4, unofficial]. Stores the 3D entry/exit coordinates as RGB color values [S1, S4, unofficial].
3.  **Frame Buffer Object (FBO):** Attached to $g\_bfTexObj$ for the first pass [S4, unofficial].

**Numbered Steps:**

1.  **Pass 1: Capture Back-Face Coordinates**
    *   Bind the FBO attached to $g\_bfTexObj$.
    *   Set viewport to $(0, 0, W, H)$ [S1, unofficial].
    *   Enable face culling and set `glCullFace` to `GL_FRONT` (culling front faces, rendering only back faces) [S1, S4, unofficial].
    *   Render the bounding box using the back-face shader program.
    *   The back-face vertex shader outputs the vertex position (object coordinates) as `fragPos` [S4, unofficial].
    *   The back-face fragment shader writes `vec4(fragPos, 1.0)` to the output color attachment [S4, unofficial]. This stores the 3D entry point (or exit point, depending on camera orientation and culling definition) for each pixel covered by the back face [S1, S4, unofficial].
    *   Unbind the FBO [S1, unofficial].

2.  **Pass 2: Ray Casting and Compositing**
    *   Bind the default framebuffer.
    *   Enable face culling and set `glCullFace` to `GL_BACK` (culling back faces, rendering only front faces) [S1, S4, unofficial].
    *   Render the bounding box using the ray-casting shader program.
    *   The front-face vertex shader outputs its position as `EntryPoint` [S4, unofficial].
    *   The ray-casting fragment shader performs the following:
        a.  **Retrieve Exit Point:** Sample $g\_bfTexObj$ using normalized fragment coordinates (`gl_FragCoord.st / u_Resolution`) to retrieve the 3D exit point (`ExitPoint`) [S4, unofficial].
        b.  **Compute Ray:** Calculate ray direction $\vec{d} = \text{ExitPoint} - \text{EntryPoint}$ and length $L = \|\vec{d}\|$ [S4, unofficial].
        c.  **Edge Case:** If $\text{EntryPoint} == \text{ExitPoint}$ (ray misses or is tangent), discard the fragment to avoid division by zero [S4, unofficial].
        d.  **Initialize:** Set step vector $\vec{s} = \vec{d} / N$ (where $N$ is the number of steps) or $\vec{s} = \text{normalize}(\vec{d}) \times \text{stepSize}$ [S1, S4, unofficial]. Initialize accumulated color $C_{acc} = (0,0,0,0)$ and opacity $\alpha_{acc} = 0$ [S4, unofficial]. Set current position $\mathbf{p} = \text{EntryPoint}$ [S4, unofficial].
        e.  **Ray Marching Loop:** For $i = 0$ to $N-1$:
            i.  Sample volume value $v = V(\mathbf{p})$ [S4, unofficial].
            ii. Map $v$ to color $C_i$ and opacity $\alpha_i$ using the transfer function [S4, unofficial].
            iii. Apply compositing:
            $$
            C_{acc} = C_{acc} + (1 - \alpha_{acc}) \alpha_i C_i
            $$
            $$
            \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \alpha_i
            $$
            iv. Advance position: $\mathbf{p} = \mathbf{p} + \vec{s}$ [S4, unofficial].
            v.  **Termination:** Break if $\alpha_{acc} \ge 1.0$ (early ray termination) or if $\mathbf{p}$ exceeds the volume bounds [S1, S4, unofficial].
        f.  **Final Compositing:** If the loop completes without $\alpha_{acc}=1$, composite with background color $C_{bg}$:
        $$
        C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}
        $$
        g.  Output $C_{final}$ [S4, unofficial].

**Pseudocode (GLSL-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Pass 1: Backface Shader
// Vertex Shader
in vec3 VerPos;
uniform mat4 MVP;
out vec3 fragPos;

void main() {
    fragPos = VerPos;
    gl_Position = MVP * vec4(VerPos, 1.0);
}

// Fragment Shader
in vec3 fragPos;
layout (location = 0) out vec4 gExisPoints;

void main() {
    gExisPoints = vec4(fragPos, 1.0);
}

// Pass 2: Raycasting Shader
// Vertex Shader
in vec3 VerPos;
uniform mat4 MVP;
out vec3 EntryPoint;

void main() {
    EntryPoint = VerPos;
    gl_Position = MVP * vec4(VerPos, 1.0);
}

// Fragment Shader
in vec3 EntryPoint;
uniform sampler2D u_ExitPoints;
uniform sampler3D u_VolumeTex;
uniform sampler1D u_TransferFunc;
uniform vec2 u_Resolution;
uniform float u_StepSize; // Or number of steps

layout (location = 0) out vec4 FragColor;

void main() {
    // Retrieve exit point from Pass 1 texture
    vec3 exitPoint = texture(u_ExitPoints, gl_FragCoord.st / u_Resolution).xyz;
    
    // Discard if entry equals exit (no intersection)
    if (EntryPoint == exitPoint) {
        discard;
    }
    
    // Compute ray direction and length
    vec3 dir = exitPoint - EntryPoint;
    float len = length(dir);
    
    // Compute step vector
    // Note: u_StepSize can be interpreted as fixed step size or normalized step
    // Here we assume u_StepSize is the physical step size along the ray
    vec3 deltaDir = normalize(dir) * u_StepSize;
    
    vec3 voxelCoord = EntryPoint;
    vec4 colorAcum = vec4(0.0);
    float alphaAcum = 0.0;
    
    // Fixed loop limit for demonstration; typically determined by len / u_StepSize
    for(int i = 0; i < 1600; i++) {
        // Sample volume
        float intensity = texture(u_VolumeTex, voxelCoord).x;
        
        // Map to color/opacity
        vec4 colorSample = texture(u_TransferFunc, intensity);
        
        if (colorSample.a > 0.0) {
            // Apply transfer function alpha adjustment if needed
            // colorSample.a = 1.0 - pow(1.0 - colorSample.a, .1);
            
            // Front-to-back compositing
            colorAcum.rgb += (1.0 - colorAcum.a) * colorSample.rgb * colorSample.a;
            colorAcum.a += (1.0 - colorAcum.a) * colorSample.a;
        }
        
        // Advance
        voxelCoord += deltaDir;
        
        // Termination conditions
        if (colorAcum.a >= 1.0) {
            break;
        }
        
        // Check bounds (optional if step size is small enough or loop limit is sufficient)
        // if (any(voxelCoord < 0.0) || any(voxelCoord > 1.0)) break;
    }
    
    // Composite with background
    vec4 bgColor = vec4(1.0, 1.0, 1.0, 1.0);
    FragColor = colorAcum + (1.0 - colorAcum.a) * bgColor;
}
```

**Parameters:**
*   `uSteps` / `u_StepSize`: Number of sampling steps or step size. Typical `uSteps`: 256 [S1, unofficial]. Typical `DELTA` (step size): 0.005 [S3, unofficial].
*   `uOpacityVal`: Opacity scaling factor. Typical value: 4.0 [S1, unofficial].
*   `uMinGrayVal` / `uMaxGrayVal`: Volume value thresholds. Typical: 0.001 to 1.0 [S1, unofficial].
*   `uAbsorptionModeIndex`: Rendering mode switch (0: Simple accumulation, 1: Alpha blending, 2: Max projection) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zooming:** Rendering only front faces causes clipping when the camera zooms in, as front faces project behind the camera. Rendering back faces avoids this [S2, unofficial].
*   **Tangent/Missed Rays:** If the ray is tangent to the box or misses it, the entry and exit points may be identical or invalid. The shader must discard such fragments to prevent division by zero or infinite loops [S4, unofficial].
*   **Precision:** Using `GL_RGBA16F` for the back-face texture helps maintain precision for coordinate storage [S4, unofficial].
*   **Loop Limits:** GLSL may require large constant loop limits (e.g., 4095) with explicit break conditions based on uniforms [S1, unofficial].

## Subtopics

### [Bounding Box Raymarching](2071-bounding-box-raymarching.md)
A variant that explicitly utilizes a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints. It differs by often computing ray directions from geometry attributes or camera matrices directly in the fragment shader.

### [Entry Point Adjustment](7406-entry-point-adjustment.md)
A technique to refine the entry points determined by back-face rasterization, often to handle non-convex volumes or improve sampling accuracy at boundaries.

## Operating characteristics
*   **Time Complexity:** $O(W \times H \times N)$, where $W, H$ are viewport dimensions and $N$ is the number of steps per ray. The two-pass approach adds a constant overhead for the back-face rendering pass.
*   **Memory Complexity:** $O(W \times H)$ for the back-face texture storage [S4, unofficial].
*   **Performance:** Efficiently culls pixels outside the volume, reducing fragment shader invocations [S2, unofficial]. Early ray termination further reduces average computation time [S1, S2, S3, S4, unofficial].
*   **Strengths:** Simple implementation using standard rasterization hardware; avoids complex ray-box intersection math in the fragment shader for boundary determination [S3, S4, unofficial]; handles zooming correctly by rendering back faces [S2, unofficial].
*   **Limitations:** Requires two render passes; limited to convex bounding boxes (typically axis-aligned); precision of entry/exit points depends on texture format and rasterization accuracy [S4, unofficial].
*   **When to Use:** Suitable for real-time volume rendering of convex volumes on GPUs with limited ray-tracing hardware.
*   **When Not to Use:** For non-convex volumes or when single-pass performance is critical and intersection math is acceptable [S5, unofficial].

## Background
The method originated as a solution to the inefficiency of full-screen ray casting, where fragment shaders are invoked for pixels outside the volume [S2, unofficial]. It leverages the GPU's rasterization pipeline to determine ray bounds, a technique popularized in early GPU-based volume rendering implementations [S1, S4, unofficial]. It is widely used in educational examples and early WebGL volume renderers [S2, S3, unofficial].

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S3] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S4] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S5] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
