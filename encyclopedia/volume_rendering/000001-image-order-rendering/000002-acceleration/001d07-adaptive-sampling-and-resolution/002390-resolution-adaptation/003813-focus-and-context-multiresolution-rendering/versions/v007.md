# Focus and Context Multiresolution Rendering
Also called: Interactive lenses multiresolution, Time-critical multiresolution rendering

## Overview
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications [S1, unofficial]. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly separating the scene into two distinct rendering passes with different resolutions and rendering models, rather than adjusting a single global resolution or using hierarchical data structures. The goal is not seamless integration but a tradeoff between quality and speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function, a camera view, and a defined "focus" region (e.g., a data probe or specific spatial subset).
*   **Output:** A composite image where the focus region is rendered at high resolution and the context region at lower resolution.

**Data Structures:**
The method utilizes two distinct rendering configurations for the same underlying volume data. The specific parameters used in the source evaluation are detailed below [S1, unofficial]:

| Parameter | Focus Configuration | Context Configuration |
| :--- | :--- | :--- |
| **Axis Volume** | $4 \times 256^3$ | $1 \times 128^3$ |
| **Lookup Table Size** | $5 \times 256^2$ | $4 \times 128^2$ |
| **Rendering Model** | Local lighting | Unshaded |
| **Sampling Rate** | 4 | 1 |
| **Viewport Size** | 100% | 50% |

*Note: All textures had 8-bit components [S1, unofficial].*

**Steps:**
1.  **Decomposition:** Divide the rendering task into two passes: one for the focus region and one for the context region. The focus is placed closest to the user, mimicking typical interface usage [S1, unofficial].
2.  **Context Rendering:**
    *   Render the context volume into a smaller viewport (e.g., 50% of the full viewport dimensions) [S1, unofficial].
    *   Use a simpler rendering model (unshaded) and lower sampling rate (1) to reduce fragment count [S1, unofficial].
    *   If the context is transparent (used for orientation), it can be sampled sparsely if the transfer function contains only low-frequency components [S1, unofficial].
    *   Apply a Gaussian filter to the resulting context image to reduce aliasing artifacts caused by the upsampling [S1, unofficial].
3.  **Focus Rendering:**
    *   Render the focus volume into the full viewport (100%) or the remaining area.
    *   Use high-resolution data, a larger lookup table, local lighting, and a higher sampling rate (e.g., 4) [S1, unofficial].
4.  **Compositing:** Enlarge and composite the context image into the final frame, overlaying it with the high-resolution focus image. The integration does not need to be seamless [S1, unofficial].

**Parameters:**
*   **Context Rendering Mode:** Can be transparent (for orientation) or opaque (for external view) [S1, unofficial].
*   **Viewport Scaling:** The context viewport size is a reduction factor $n \in (0.0, 1.0]$. In the source, $n=0.5$ was used [S1, unofficial].
*   **Sampling Rate:** Focus uses a higher rate (4) than context (1) [S1, unofficial].

**Pitfalls:**
*   **Overhead:** Performance overhead comes from rendering slices multiple times and setting up clip textures and the depth buffer for each pass [S1, unofficial].
*   **Artifacts:** Without the Gaussian filter, aliasing artifacts may appear in the context region due to the resolution mismatch [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The method reduces the number of fragments processed in the fragment pipeline for the context region by rendering it at a lower resolution. The speedup depends on the reduction factor of the viewport and volume size.

**Measured Performance:**
Tests were conducted on a 2.0GHz Pentium 4 with 2GB memory and an ATI Radeon 9800 Pro graphics card [S1, unofficial]. Acceleration techniques like early ray termination and empty space skipping were turned off to isolate the impact of the multiresolution decomposition [S1, unofficial].

*   **512^2 Viewport:**
    *   100% Focus Size: 3.39 FPS
    *   50% Volume Size: 4.22 FPS
    *   25% Volume Size: 7.55 FPS
    *   12.5% Volume Size: 12.99 FPS
    *   6.25% Volume Size: 20.87 FPS
*   **1024^2 Viewport:**
    *   100% Focus Size: 1.06 FPS
    *   50% Volume Size: 1.28 FPS
    *   25% Volume Size: 2.31 FPS
    *   12.5% Volume Size: 3.97 FPS
    *   6.25% Volume Size: 6.33 FPS

**Scaling:**
Reducing the number of fragments required to render the focus more aggressively achieves 2-6 times faster rendering rates than using a single high-resolution mode [S1, unofficial]. The attainable speedup does not depend on the transfer functions, only on the rendering parameters used [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Effective for balancing limited hardware resources; improves interactivity in time-critical applications; simple to implement as a generic approach [S1, unofficial].
*   **Limitations:** Introduces overhead from multiple rendering passes and buffer setup; not designed for seamless visual integration; performance variability exists depending on slicing direction on certain hardware (e.g., ATI cards) [S1, unofficial].

## Background
This technique was presented in the unpublished work "A Focus and Context Interface for Interactive Volume Rendering" by I. Kits et al. [S1, unofficial]. It addresses the limitations of hardware-accelerated volume rendering where performance depends on fragment count, work per fragment, and cache utilization. The method builds on the principle that multiresolution techniques can use subsampled data blocks to increase cache coherency and reduce sampling rates, but extends this by explicitly balancing resources between focus and context regions [S1, unofficial].

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
