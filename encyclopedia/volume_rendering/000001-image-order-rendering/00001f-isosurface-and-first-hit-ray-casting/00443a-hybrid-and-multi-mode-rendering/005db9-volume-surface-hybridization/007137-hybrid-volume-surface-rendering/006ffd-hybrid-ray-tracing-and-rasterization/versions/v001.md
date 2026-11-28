# Hybrid Ray Tracing and Rasterization
Also called: Hybrid RT, Ray Tracing and Hybrid Rendering, Real-time Path Tracing / Hybrid RT, リアルタイムパストレーシング / ハイブリッドRT
## Overview
Hybrid Ray Tracing and Rasterization is a rendering technique that combines the accuracy of ray tracing with the performance of rasterization-based techniques to achieve high visual fidelity and real-time interaction [S1, S2, unofficial]. It addresses the trade-off between the speed of rasterization and the physical accuracy of ray tracing by using rasterization for the primary image pass and ray tracing to augment specific effects such as reflections, shadows, and global illumination [S2, unofficial].
## How it works
The method operates in two distinct phases, leveraging modern GPU hardware capabilities introduced with architectures like NVIDIA RTX and APIs like DirectX Raytracing (DXR) [S2, unofficial].

**Inputs and Outputs:**
*   **Inputs:** A scene graph containing geometry, materials, and lighting; a rasterization pipeline configuration; a ray tracing acceleration structure (e.g., BVH); and a denoising algorithm (often AI-based) [S2, unofficial].
*   **Outputs:** A final composite image with physically plausible lighting effects [S1, S2, unofficial].

**Data Structures:**
*   **Rasterization Buffers:** Standard framebuffers for color, depth, and normals [G].
*   **Acceleration Structure:** A Bounding Volume Hierarchy (BVH) or similar structure for efficient ray-scene intersection tests, required for the ray tracing pass [G].

**Steps:**
1.  **Rasterization Pass:** Render the scene using standard rasterization techniques to generate the base color, depth, and other auxiliary buffers. This step handles the majority of the geometry and provides the primary lighting contribution [S2, unofficial].
2.  **Ray Tracing Pass:** For specific effects (reflections, shadows, global illumination), cast rays from the pixel coordinates into the scene using the acceleration structure. Calculate the intersection points and compute lighting contributions for these specific paths [S2, unofficial].
3.  **Compositing:** Combine the results from the rasterization pass and the ray tracing pass. The ray-traced effects are typically blended or overlaid onto the rasterized base image [S2, unofficial].
4.  **Denoising:** Apply an AI-based denoising algorithm to the ray-traced components to reduce noise, which is inherent in real-time path tracing due to limited sample counts [S2, unofficial].

**Parameters:**
*   **Ray Depth:** Typically limited to 1-3 bounces for real-time performance [G].
*   **Sample Count:** Low sample counts per pixel (e.g., 1-4) for ray-traced effects, relying on denoising [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Hardware Dependency:** The technique is highly dependent on hardware support for ray tracing (e.g., RT cores) [S2, unofficial].
*   **Power and Thermal Constraints:** High computational load can lead to significant power consumption and thermal issues, limiting its use in mobile or thermally constrained devices [S2, unofficial].
*   **Optical Inconsistency:** While improved, hybrid methods may still exhibit optical inconsistencies compared to full path tracing, particularly in complex lighting scenarios [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the rasterization pass, which is $O(N)$ where $N$ is the number of pixels and primitives. The ray tracing pass adds overhead proportional to the number of rays cast and the complexity of the acceleration structure [G].
*   **Memory Complexity:** Requires additional memory for acceleration structures and auxiliary buffers for ray tracing effects [G].
*   **Performance:** Offers a balance between quality and speed, making it a practical solution for real-time applications like games [S2, unofficial]. However, it is significantly more demanding than pure rasterization [S2, unofficial].
*   **Strengths:** High visual fidelity for specific effects; real-time performance achievable on modern hardware [S2, unofficial].
*   **Limitations:** Hardware dependency; potential for optical inconsistencies; high power consumption [S2, unofficial].
*   **When to Use:** Real-time applications requiring high-quality lighting effects, such as modern video games [S2, unofficial].
*   **When Not to Use:** Applications requiring full physical accuracy (use full path tracing) or those running on hardware without ray tracing support [S2, unofficial].

## Background
The hybrid approach emerged as a practical compromise between the high quality of offline rendering and the speed requirements of real-time applications [S2, unofficial]. With the introduction of NVIDIA RTX and DirectX Raytracing (DXR) in 2018, hybrid rendering became the mainstay in the gaming industry [S2, unofficial]. Since 2023, efforts have shifted towards "Real-time Path Tracing," which attempts to run full Monte Carlo path tracing in real-time, supported by AI denoising, with examples like NVIDIA Portal RTX [S2, unofficial].
## Sources
[S1] Llama version of Ct Volume Rendering, https://www.flyriver.com/l/ct-volume-rendering, fetched 2026-10-29 (unofficial)
[S2] 3Dレンダリング方式についてchatGPTに聞いてみた｜ほねぷー, https://note.com/honepoo/n/nc610518091f7, fetched 2026-10-30 (unofficial)
