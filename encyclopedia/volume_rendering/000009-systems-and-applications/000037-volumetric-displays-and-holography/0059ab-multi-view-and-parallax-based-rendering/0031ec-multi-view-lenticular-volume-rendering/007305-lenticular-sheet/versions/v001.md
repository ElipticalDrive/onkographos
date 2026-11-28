# Lenticular Sheet

Also called: Lenticular Array, Lenticular Lens Array

## Overview
A lenticular sheet is a refractive optical component consisting of a linear array of thick plano-convex cylindrical lenses, known as lenticules, used in multiview autostereoscopic displays to direct light from a 2D screen into distinct viewing zones [S1, unofficial]. It solves the problem of generating multi-view stereoscopic images without requiring viewers to wear glasses, functioning as a transparent, high-efficiency alternative to parallax barriers [S1, unofficial]. Unlike parallax barriers which block light, lenticular sheets refract it, offering better brightness and higher potential resolution, though it suffers from limited per-view resolution (1/N of the display resolution for N views) and strict alignment requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D LCD screen displaying a spatially multiplexed image; a lenticular sheet placed in front of the screen.
*   **Output:** A multiview 3D image where different viewing angles perceive different perspectives (views) of the volumetric or stereoscopic content.

**Data Structures and Layout:**
The lenticular sheet consists of a periodic array of cylindrical lenses. The key geometric parameter is the lens pitch (width of one lenticule). The 2D LCD screen has a pixel grid. The system requires precise physical alignment between the lens pitch and the LCD pixel columns.

**Steps:**
1.  **Spatial Multiplexing:** Divide the horizontal resolution of the 2D display screen among $N$ distinct views. For every $N$ vertical pixel columns on the LCD, one column is assigned to a single view [S1, unofficial]. For example, in a 5-view system, every 5th column carries data for the same view [S1, unofficial].
2.  **Alignment:** Physically align the lenticular lenses with the vertical pixel columns of the 2D LCD screen. The lenses must be parallel to the vertical pixel boundaries to ensure correct refraction [S1, unofficial].
3.  **Refraction:** Light from each pixel column passes through the corresponding cylindrical lens. The lens refracts the light rays so that they converge into specific viewing zones (viewing windows) in space [S1, unofficial].
4.  **Viewing:** A viewer positions their eyes within a correct view zone. Each eye receives light from a specific subset of pixels, creating a stereoscopic pair. Head movement allows the viewer to experience motion parallax as they move between view zones [S1, unofficial].

**Formulas and Parameters:**
*   Let $N$ be the number of views.
*   Let $W_{screen}$ be the horizontal resolution of the display in pixels.
*   The horizontal resolution per view, $W_{view}$, is:
    $$ W_{view} = \frac{W_{screen}}{N} $$
*   The pitch of the lenticular array ($P_{lens}$) must match the width of the group of pixels assigned to one view ($P_{pixel} \times N$, where $P_{pixel}$ is the pixel pitch) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Resolution Loss:** The resolution of an individual view is essentially $1/N$ of the original display resolution [S1, unofficial].
*   **Alignment Sensitivity:** Misalignment between the lenticular sheet and the LCD pixel columns causes cross-talk between views and image distortion [S1, unofficial].
*   **Image Flips:** Incorrect alignment or viewing position can cause one eye to see the image intended for the other eye, resulting in incorrect stereo perception [S1, unofficial].
*   **Picket Fence Problem:** Similar to parallax barriers, lenticular displays can suffer from the "picket fence" effect, where the viewer sees the gaps or boundaries between views [S1, unofficial].
*   **Limited Viewing Windows:** The number of distinct viewing zones is limited by the number of views $N$ and the optical design [S1, unofficial].

## Subtopics

### Eye-Tracking Lenticular Display
A display variant using an array of lenses (where lenslet width > pitch), an LCOS projector, and an LCD panel, where the lens array steers light from the projector to the viewer's eye location determined by an eye-tracking system [S1, unofficial]. The LCOS projector image consists of a series of dots designed for the lens array to focus onto the viewer's eye locations [S1, unofficial]. The primary pitfall is low optical efficiency, as only a small portion of light from the LCOS projector is used for directional backlight [S1, unofficial].

### Liquid Crystal Lenses
Uses liquid crystal (LC) material whose shape is controlled to serve as an optical lens in front of a Spatial Light Modulator (SLM), directing light beams to desirable directions in real time [S1, unofficial]. Voltage applied between ITO layers forms the shape of LC cells into an array of optical lenses equivalent to a lenticular lens array [S1, unofficial]. Multiple electrodes (MeD LC Lens) can be used to better control the lens shape than conventional two-electrode lenses [S1, unofficial]. The LC lens shape can be dynamically changed at the image refreshing rate to allow active scanning of light beams for multiview generation, projecting high frame rate images from the SLM in a time-sequential fashion [S1, unofficial]. A major pitfall is irregularity of LC alignment in the boundary region, causing serious cross-talk and image deterioration [S1, unofficial]. Unlike fixed lenticular displays, it is electronically switchable between 2D and 3D modes, preserving native SLM resolution in 2D mode [S1, unofficial].

### Slanted Lenticular Layer
A technique to compensate for resolution loss by slanting the lenticular lens structure or rearranging color filter pixels to distribute resolution loss into both horizontal and vertical directions [S1, unofficial]. The LCD is located in the focal plane of the lenticular sheet, mapping horizontal position on the LCD to viewing angle [S1, unofficial]. Points on specific lines (e.g., line A-A) are aligned to direct specific views (e.g., view 3) in a given direction [S1, unofficial]. The flipping effect is reduced by ensuring predominant view contribution with some contribution from adjacent views (e.g., view 3 predominates with some contribution from view 2 on line A-A) [S1, unofficial]. This technique is used in 3D TV products by Vizio, Stream TV Networks, Alioscopy, RealD, Philips, Sharp, Toshiba, and TLC [S1, unofficial].

### High-Resolution Multiview LCD with Lenticular/Barrier
A technique to address resolution loss by splitting conventional square pixels into $N$ portions, each covered by special lenticular arrays directing RGB portions to different view directions [S1, unofficial]. The conventional square pixel is split into $N$ portions, and special lenticular arrays are designed to cover each portion with different optics [S1, unofficial]. Each portion of RGB is directed toward a different view direction [S1, unofficial]. While it suffers from the conventional drawback that horizontal resolution for each view is only $1/N$ of the SLM's native resolution in HPO systems, this design allows each view to have the pixel resolution of the original LCD, unlike conventional lenticular displays [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Lenticular-based displays offer better brightness and higher possible resolution than parallel-barrier-based displays [S1, unofficial].
*   **Scaling:** Resolution per view scales inversely with the number of views ($1/N$) [S1, unofficial].
*   **Strengths:** Utilizes existing 2D screen fabrication infrastructure; implementation is relatively simple and low cost; higher optical efficiency than parallax barriers due to transparency [S1, unofficial].
*   **Limitations:** Limited resolution per view; significant effort required for alignment; cross-talk between views and image flips; picket fence problem; limited number of viewing windows [S1, unofficial].
*   **When to use:** When high brightness and simple hardware implementation are prioritized over high per-view resolution, and when the display can be viewed from a fixed or limited set of positions [S1, unofficial].
*   **When not to use:** When high per-view resolution is critical, or when the viewing environment requires dynamic, eye-tracked positioning without additional complexity [S1, unofficial].

## Background
The concept of using a one-dimensional (1D) lenticular array for stereoscopic display was patented by Hess in 1912 [S1, unofficial]. The technique is part of the broader field of refraction-based multiview 3D display techniques [S1, unofficial]. It is widely used in commercial 3D TV products, with manufacturers such as Vizio, Stream TV Networks, Alioscopy, RealD, Philips, Sharp, Toshiba, and TLC adopting slanted lenticular variants [S1, unofficial]. Prototypes of high-resolution multiview displays using specialized LCD and lenticular arrays have been developed by NLT for industrial applications [S1, unofficial].

## Sources
[S1] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
