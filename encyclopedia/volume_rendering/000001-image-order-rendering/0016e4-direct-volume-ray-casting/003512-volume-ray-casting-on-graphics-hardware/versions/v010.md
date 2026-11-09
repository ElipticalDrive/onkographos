# Volume Ray-Casting on Graphics Hardware

Also called: Stream model for volumetric ray-casting

## Overview
This method implements direct volume ray casting on programmable graphics hardware (specifically DirectX 9-level architectures like the ATI 9700) using a multi-pass stream model [S1, unofficial]. It solves the performance bottleneck of CPU-based ray casting by exploiting the intrinsic parallelism of fragment processors and efficient inter-pass communication via texture render targets [S1, unofficial]. It is distinguished from single-pass hardware implementations by its explicit use of an early z-test mechanism to terminate ray traversal, allowing it to skip empty space and stop processing once an opacity threshold or iso-surface is reached [S1, unofficial].

## How it works
The method operates as a multi-pass pipeline where the output of one pass serves as the input texture for the next. It relies on specific hardware features: Pixel Shader 2.0, per-fragment texture fetches (up to 8 textures, limited dependent fetches), texture render targets (2D textures aligned with the viewport, supporting floating-point formats for negative values), texture coordinate generation, per-fragment arithmetic, and depth replace capabilities [S1, unofficial].

### Inputs and Outputs
- **Input**: A 3D scalar volume texture, a pre-computed ray direction texture (storing the direction in local texture coordinates for each pixel), and a transfer function.
- **Output**: A final 2D color image. Intermediate results (accumulated color, opacity, and ray direction) are stored in 2D floating-point textures between passes [S1, unofficial].

### Data Structures
1.  **Volume Texture**: 3D texture containing scalar density values.
2.  **Ray Direction Textures**: Two 2D textures storing the $x$ and $y$ components of the ray direction vector in local texture space for each pixel. These are computed prior to ray traversal and remain constant across passes [S1, unofficial].
3.  **Accumulation Textures**: 2D floating-point textures used to store the accumulated color $C'$ and accumulated opacity $\alpha'$ for each pixel. These are updated in each main pass and read in the subsequent pass [S1, unofficial].
4.  **Z-Buffer**: Used for the early z-test. The depth value is manipulated to control fragment processing [S1, unofficial].

### Steps
1.  **Initialization**: Compute the ray direction for each pixel in local texture coordinates. Store these directions in two 2D textures [S1, unofficial]. Initialize the accumulation textures with zero color and zero opacity [S1, unofficial].
2.  **Main Ray Casting Passes**: Perform a fixed number of rendering passes. In each pass:
    a.  Bind the current accumulation textures as inputs.
    b.  Bind the volume texture and ray direction textures.
    c.  Execute a pixel shader that performs one step of ray traversal:
        i.  Retrieve the current accumulated color $C'_{i-1}$ and opacity $\alpha'_{i-1}$ from the accumulation textures using the current pixel coordinates [S1, unofficial].
        ii. Retrieve the ray direction and compute the current sample position in the volume [S1, unofficial].
        iii. Sample the volume at the current position to get the scalar value $d_i$ [S1, unofficial].
        iv. Apply the transfer function to obtain color $C_i$ and opacity $\alpha_i$ [S1, unofficial].
        v.  Update the accumulated color and opacity using the front-to-back compositing equations:
            $$
            C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
            $$
            $$
            \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
            $$
        vi. Write the updated $C'_i$ and $\alpha'_i$ to the output accumulation textures [S1, unofficial].
3.  **Termination Check Pass**: Between main passes, execute an additional pass to check for termination:
    a.  Read the accumulated opacity $\alpha'$ from the accumulation texture [S1, unofficial].
    b.  Compare $\alpha'$ against a specified threshold (or check if an iso-surface was hit, if applicable) [S1, unofficial].
    c.  If the condition is met (threshold exceeded or iso-surface hit), set the fragment's depth value ($z$) to the maximum possible value using the depth replace feature [S1, unofficial].
    d.  Otherwise, set the depth value to zero [S1, unofficial].
4.  **Early Z-Test Filtering**: Configure the graphics pipeline's z-test to `GREATER`. Because the termination check pass sets the depth to the maximum for terminated rays, subsequent main passes will fail the z-test for these fragments. The early z-test discards these fragments before the pixel shader executes, effectively skipping empty space and terminating the ray [S1, unofficial].
5.  **Final Pass**: After all main passes, the final accumulated color and opacity in the accumulation textures represent the rendered image. The final color is typically computed as $C_{final} = C'_{N} + (1 - \alpha'_{N}) C_{background}$, where $C_{background}$ is the background color [S1, unofficial]. If an iso-surface was hit, a final pass may restrict gradient and shading computations to those intersection points [S1, unofficial].

### Parameters
-   **Opacity Threshold**: The value at which ray traversal terminates. Not given in the sources.
-   **Number of Main Passes**: Fixed number of ray casting steps. Not given in the sources.
-   **Hardware**: ATI 9700 graphics card [S1, unofficial].
-   **API**: Pixel Shader 2.0 (DirectX 9-level) [S1, unofficial].

### Edge Cases and Pitfalls
-   **Dependent Texture Fetches**: The ATI 9700 has a limited number of dependent texture fetches. The algorithm must minimize these to avoid performance degradation [S1, unofficial].
-   **Floating Point Precision**: Floating-point textures are required to store negative values and high dynamic range accumulation results. Precision issues may arise if the hardware does not support sufficient bit depth [S1, unofficial].
-   **Early Z-Test Dependency**: The early z-test only works if all other per-fragment tests (like stencil or alpha tests) are disabled or configured to pass, as the early z-test is applied before the shader executes [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: The number of operations per pixel is proportional to the number of main passes, but early termination reduces the average number of passes for opaque regions. Performance gains of up to a factor of 3 were demonstrated on the ATI 9700 for typical volumetric data sets [S1, unofficial].
-   **Memory Complexity**: Requires storage for the volume texture, ray direction textures, and accumulation textures. The accumulation textures are 2D and aligned with the viewport [S1, unofficial].
-   **Strengths**: Exploits hardware parallelism; efficient empty space skipping and early ray termination via early z-test; suitable for real-time rendering on DirectX 9-level hardware [S1, unofficial].
-   **Limitations**: Requires specific hardware features (texture render targets, depth replace, early z-test); limited by the number of dependent texture fetches on older hardware; fixed number of main passes may lead to over-sampling in some cases [S1, unofficial].
-   **When to Use**: Real-time volume rendering on programmable graphics hardware with support for Pixel Shader 2.0 and texture render targets [S1, unofficial].
-   **When Not to Use**: On hardware lacking texture render targets or early z-test support; when adaptive stepping is required without multi-pass overhead [S1, unofficial].

## Background
The method was proposed in a 2003 paper titled "Acceleration Techniques for GPU-based Volume Rendering" by the authors associated with the Technical University of Munich (TUM) [S1, unofficial]. It was developed to address the challenge of achieving real-time or interactive rendering of large volumetric scalar fields, which was not feasible with CPU-based methods at the time [S1, unofficial]. The work demonstrates the effectiveness of the stream model on the ATI 9700 graphics card, leveraging the Pixel Shader 2.0 API [S1, unofficial].

## Sources
[S1] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
