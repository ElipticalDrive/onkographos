# Multi-view Lenticular Volume Rendering

Also called: GPU-accelerated volume rendering to 3D lenticular displays

## Overview
Multi-view Lenticular Volume Rendering is a hardware-accelerated method for generating stereoscopic images of volumetric datasets (typically medical) for display on multi-view lenticular screens without glasses [S1, unofficial]. It differs from single-view volume rendering by computing $N$ distinct perspective views from offset focal spots and compositing them into a single interlaced bitmap where each sub-pixel (R, G, B) samples from a different view [S1, unofficial]. It is distinct from holographic methods by using standard Direct Volume Rendering (DVR) with alpha blending rather than wavefront calculation [S1, unofficial].

## How it works
**Inputs:** Volumetric dataset (scalar field), lenticular display geometry (focal distance, lens pitch), number of views $N$.
**Outputs:** Interlaced bitmap for the LCD panel.

**Data Structures:**
1.  **Bricks:** The volumetric dataset is divided into 3D blocks ("bricks") to fit within the GPU's texture memory and align with its memory architecture [S1, unofficial].
2.  **Texture0 (Static View Map):** A 2D texture map storing the view index for every sub-pixel (R, G, B) of the output image. This is pre-calculated based on lenticular lens geometry and remains constant [S1, unofficial].
3.  **Texture1 (View Stack):** A single wide rectangular 2D texture map storing the rendered images of all $N$ views vertically stacked next to each other [S1, unofficial].

**Algorithm Steps:**

1.  **Brick Loading:** Load the volumetric data bricks into GPU texture memory [S1, unofficial].
2.  **View Rendering Loop:** For each view $n$ from $0$ to $N-1$:
    a.  **Camera Setup:** Define an asymmetric frustum perspective projection. The camera focal spot is offset horizontally. The projection coordinates $(x', y')$ for a world point $(x, y, z)$ are calculated as [S1, unofficial]:
        $$
        x' = \frac{(x - n \cdot d) \cdot f}{f - z} + n \cdot d
        $$
        $$
        y' = \frac{y \cdot f}{f - z}
        $$
        where $f$ is the focal distance, $d$ is the distance between adjacent view cameras, and $n$ is the view index [S1, unofficial].
    b.  **Direct Volume Rendering:** Render the volume using DVR. Each ray is defined by the focal spot and the pixel position. The volume rendering integral is approximated by traversing rays in back-to-front order using the over operator [S1, unofficial]:
        $$
        C_{n+1} = \alpha_n \cdot c_n + (1 - \alpha_n) \cdot C_n
        $$
        where $C_n$ is the intermediate color value, $\alpha_n$ is the opacity, and $c_n$ is the color at sample $n$ [S1, unofficial].
    c.  **Compositing:** Use standard alpha blending (via DirectX or OpenGL) to blend textured slices containing volumetric data back-to-front. Intermediate results $C_n$ are written to the frame buffer [S1, unofficial].
    d.  **Storage:** Store the resulting image for view $n$ in the corresponding vertical section of Texture1 [S1, unofficial].
3.  **Final Compositing:** Parse the output image pixels. For each pixel, sample the Red, Green, and Blue components from Texture1 using the view indices provided by Texture0. This creates the final interlaced image displayed on the lenticular screen [S1, unofficial].

**Parameters:**
*   **Number of views ($N$):** The specific implementation uses 9 views, but the method applies to any $N$ [S1, unofficial].
*   **Resolution:** Each view has an effective resolution of $1/N$ of the LCD matrix resolution in the horizontal direction [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Limits:** Large datasets exceeding texture memory must be correctly partitioned into bricks [S1, unofficial].
*   **Precision:** The asymmetric frustum calculation requires careful handling of depth ($z$) to avoid division by zero or near-zero if $z \approx f$ [S1, unofficial].
*   **Sub-pixel Alignment:** The view mapping in Texture0 must precisely match the physical lenticular lens geometry to ensure correct stereoscopic separation [S1, unofficial].

## Operating characteristics
*   **Performance:** Capable of up to 15 frames per second for a $256^3$ voxel dataset on a nine-view lenticular display [S1, unofficial].
*   **Complexity:** Time complexity is proportional to $N \times \text{Volume Size} \times \text{Samples per Ray}$. The use of lower resolution per view ($1/N$ horizontal resolution) partially compensates for the $N$ render passes [S1, unofficial].
*   **Strengths:** Interactive frame rates for large datasets (>100 MB) due to GPU acceleration of both rendering and compositing; no glasses required for viewers [S1, unofficial].
*   **Limitations:** Limited to the number of views supported by the display hardware; resolution per view is reduced compared to a full-resolution single view [S1, unofficial].

## Background
This method was proposed by Daniel Ruijters from Philips Medical Systems to address the computational demands of generating multi-view stereoscopic images of large volumetric medical datasets [S1, unofficial]. It leverages the SIMD capabilities of GPUs to perform Direct Volume Rendering and compositing in hardware, enabling interactive manipulation of 3D data on lenticular displays [S1, unofficial].

## Sources
[S1] GPU-accelerated volume rendering to 3D lenticular displays, https://lirias.kuleuven.be/bitstream/123456789/235668/1/GPU_Volume_Rendering_to_Lenticular_Display7.pdf, fetched 2026-10-27 (unofficial)
