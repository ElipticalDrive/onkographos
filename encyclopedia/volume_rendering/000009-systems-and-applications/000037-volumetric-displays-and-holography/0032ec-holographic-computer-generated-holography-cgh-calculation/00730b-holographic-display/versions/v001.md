# Holographic display

Also called: CGH Display, Computer-Generated Hologram (CGH) systems, Computer-Generated Holographic Display

## Overview
A holographic display is a 3D imaging technique that reproduces the wavefield of a three-dimensional scene in physical space by modulating coherent light [S1][S3, unofficial]. Unlike stereoscopic or parallax-based displays, it records and replays all characteristics of light waves (phase, amplitude, and wavelength), allowing an imaging system to see a 3D image of an object even when the object is absent [S2, unofficial]. It is distinguished from its siblings by its ability to provide true wavefront reconstruction, supporting natural focus cues and 360-degree viewing, though it faces significant hardware constraints regarding pixel pitch and data volume [S1][S3, unofficial].

## How it works
The method relies on the superposition of wavefields calculated from a target scene and displayed on a Spatial Light Modulator (SLM) [S3, unofficial]. The process involves encoding a 2D scene as a complex amplitude transparency, modulating coherent light with an SLM, and reproducing the 3D wavefield in space [S1].

**Inputs and Outputs:**
*   **Input:** A target 3D scene (represented as voxels, points, or primitives) and the parameters of the reconstruction light source (wavelength $\lambda$).
*   **Output:** A hologram pattern (phase and/or amplitude modulation) displayed on an SLM, which diffracts incident light to form a perceived 3D image at one or more depths [S3, unofficial].

**Data Structures:**
The core data structure is the **Complex Amplitude Transparency**, a 2D representation storing both amplitude and phase values of the light field for each pixel of the hologram plane [S1].
*   **Fields:**
    *   `amplitude`: Float or quantized integer representing the magnitude of the electric field.
    *   `phase`: Float or quantized integer representing the phase shift, typically in the range $[0, 2\pi]$ for phase-only SLMs [S2, unofficial].
*   **Layout:** A 2D grid of size $N_x \times N_y$, corresponding to the SLM resolution.

**Algorithm Steps:**
1.  **Scene Representation:** Represent the 3D scene as a set of $M$ elementary sources (e.g., point clouds or voxels). Each source $i$ has position $(x_i, y_i, z_i)$, amplitude $A_i$, and initial phase $\phi_{0,i}$ [S3, unofficial].
2.  **Wavefield Calculation:** Calculate the complex amplitude $U_h(x,y)$ at each pixel $(x,y)$ of the hologram plane by summing the contributions of all $M$ scene elements. The phase $\phi_i(x,y)$ of the wavefront from element $i$ arriving at the hologram pixel depends on the distance $r_i$ from the element to the pixel:
    $$ \phi_i(x,y) = k r_i + \phi_{0,i} $$
    where $k = 2\pi/\lambda$ is the wavenumber and $\lambda$ is the wavelength of the reconstruction light [S3, unofficial].
    $$ U_h(x,y) = \sum_{i=1}^{M} A_i \exp\left(i \phi_i(x,y)\right) $$
3.  **Modulation Encoding:** Extract the phase $\theta(x,y) = \arg(U_h(x,y))$ and/or amplitude $|U_h(x,y)|$ from the calculated complex field. For phase-only SLMs, the hologram is quantized to phase values $0$ to $2\pi$ [S2, unofficial].
4.  **SLM Update:** Upload the encoded hologram pattern to the SLM [S3, unofficial].
5.  **Illumination:** Illuminate the SLM with a coherent light source, such as a laser or narrowband source [S3, unofficial].
6.  **Reconstruction:** The SLM modulates the incident light, creating a diffracted image field at one or more depths, reproducing the 3D wavefield [S1][S3, unofficial].

**Parameters:**
*   **Wavelength ($\lambda$):** The wavelength of the reconstruction light. Critical for determining the wavenumber $k$ and diffraction angles [S3, unofficial].
*   **Pixel Pitch:** The physical size of the SLM pixels. To achieve high diffractive efficiency and wide viewing angles, the pixel size should be close to or below the wavelength of visible light [S2, unofficial].
*   **SLM Type:** Determines the modulation mechanism (e.g., electronically controlled LCD/LCOS, acoustically controlled AOM) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Pixel Size Limitations:** Pure holographic displays require pixel sizes smaller than 1 $\mu$m to avoid aliasing and support wide viewing angles. Current technologies (LCD, LCOS, MEMS) typically have scaling limits at 2–4 $\mu$m, resulting in narrow viewing angles (<10°) and lower visualization quality [S2, unofficial].
*   **Speckle:** Coherent illumination and SLMs introduce speckle noise, a practical difficulty in producing clean holographic images. Speckle suppression techniques are required [S3, unofficial].
*   **Data Volume:** The high resolution required for pure holography leads to multi-trillion pixel datasets, presenting challenges for acquisition, processing, and transmission [S2, unofficial].

## Subtopics

### [Holographic 3D display system based on sensitized photorefractive polymer](22652-holographic-3d-display-system-based-on-sensitized-photorefra.md)
Implements a dynamic system using sensitized photorefractive polymer (PRP) to record and reconstruct CGHs, enabling image refreshment every 2 seconds and high-brightness reconstruction visible under ambient light, unlike liquid-crystal-based volumetric displays.

## Operating characteristics
*   **Time Complexity:** The calculation of the hologram scales with the number of scene elements $M$ and hologram pixels $N$. Naive calculation is $O(M \cdot N)$, making it extremely expensive for high-resolution holograms and large scenes [G].
*   **Memory Complexity:** Requires storing the complex amplitude transparency for the entire hologram plane. For high-resolution displays (e.g., multi-trillion pixels), memory requirements are substantial [S2, unofficial].
*   **Performance:** Current CGH systems based on LCD, LCOS, or MEMS achieve relatively low visualization quality and narrow viewing angles (<10°) due to pixel pitch limits (2–4 $\mu$m) [S2, unofficial].
*   **Strengths:** Provides true wavefront reconstruction, supporting natural focus cues and 360-degree viewing, considered the ideal 3D technique [S1][S3, unofficial].
*   **Limitations:** High computational cost, large data volume, speckle noise, and hardware limitations on pixel pitch and refresh rate [S2, S3, unofficial].
*   **When to use:** Applications requiring true 3D visualization with natural depth cues, such as education, marketing, museology, and medicine [S1].
*   **When not to use:** Real-time applications with strict latency constraints or where high pixel pitch is unavoidable, unless optimized algorithms or specialized hardware (e.g., PRP) are used [S1][S2, unofficial].

## Background
Holography was invented in 1947 by Dennis Gabor and awarded the Nobel Prize in Physics in 1971 [S2, unofficial]. The term comes from the Greek *holos* (whole) and *grafe* (drawing) [S2, unofficial]. Initially, holography involved recording interference patterns on photographic plates. With the advent of computers and Spatial Light Modulators (SLMs), Computer-Generated Holography (CGH) emerged, allowing the calculation and display of holograms electronically [S3, unofficial]. Current research focuses on overcoming hardware limitations (pixel pitch, refresh rate) and computational complexity to achieve real-time, high-quality holographic displays [S2, S3, unofficial].

## Sources
[S1] Displaying three-dimensional medical visualization studies on volumetric screen, https://doi.org/10.48146/odusobiad.1625654, fetched 2026-10-26
[S2] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
[S3] Holography as Encoded Reconstruction, https://archania.org/p/technologies/displays/holography-as-encoded-reconstruction, fetched 2026-10-29 (unofficial)
