# Holographic Computer-Generated Holography (CGH) Calculation

Also called: CGH Generation, Hologram Synthesis, Computer-Generated Holography

## Overview
Holographic Computer-Generated Holography (CGH) Calculation is the computational process of synthesizing a holographic fringe pattern (complex amplitude or phase data) from a 3D scene description, such as a point cloud, voxel grid, or polygon mesh, for display on a Spatial Light Modulator (SLM) or photographic plate [S1, unofficial][S3]. It solves the problem of mapping volumetric or surface geometry into a 2D interference pattern that reconstructs the desired 3D light field when illuminated. This family is distinguished from direct volumetric display methods (like [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md) or [Photophoretic-Trap Volumetric Display](6753-photophoretic-trap-volumetric-display.md)) by generating a static or quasi-static holographic medium that encodes depth and parallax via diffraction, rather than physically moving scattering points or projecting slices [S1, unofficial][S3].

## Shared principles
All methods in this family share the fundamental goal of computing a complex optical field $U(x,y)$ on a hologram plane that, upon propagation (typically via Fourier transform or Fresnel diffraction), yields a target intensity distribution $I_r$ corresponding to the 3D object.

The core mathematical challenge is the **phase retrieval problem**: most SLMs are phase-only or amplitude-only, while the natural diffraction from a 3D object produces a complex field with both amplitude and phase. Methods generally fall into two categories:
1.  **Direct Diffraction Calculation:** Analytically or numerically summing the contributions of object elements (points, voxels, polygons) to the hologram plane using wave optics (e.g., spherical wave approximation) [S3].
2.  **Iterative Optimization:** Using algorithms like Gerchberg-Saxton or Random Phase Algorithm (ORA) to iteratively adjust the hologram phase to match a target intensity in the reconstruction plane, handling phase ambiguities [S2, 7284-computational-phase-modulation-cgh-generation.md, unofficial].

Common constraints include:
*   **Sampling:** The hologram pixel pitch must satisfy the Nyquist criterion for the highest spatial frequency in the object spectrum to avoid aliasing [S1, unofficial].
*   **Computational Load:** Direct calculation scales with the number of object elements times the number of hologram pixels ($O(N \cdot M)$), necessitating approximations (e.g., elemental holograms, hogel vectors) or hardware acceleration [S3, 7015-diffraction-specific-approach.md].
*   **Coherence:** The reconstruction assumes coherent illumination (laser light), leading to speckle noise, which some methods attempt to mitigate [S1, unofficial].

## Subtopics

### [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)
*   **What it is:** Generates CGHs for volumetric data by treating voxels as transparent objects with diffuse reflection, simulating light propagation and attenuation to preserve depth and transparency cues.
*   **Differentiation:** Unlike Maximum Intensity Projection (MIP) methods which only display max voxel values, this method calculates incident and reflected light intensities for each voxel, allowing for multi-layer and transparent object reconstruction.
*   **When to choose:** When the input data is a scalar volume (e.g., medical CT/MRI) and transparency/attenuation effects are critical for visualization.

### [Diffraction-Specific Approach](7015-diffraction-specific-approach.md)
*   **What it is:** Also known as the Hogel Vector Approach, this method bypasses direct wavefront summation by subsampling the hologram into "hogels" and representing them spectrally as "hogel vectors" derived from orthographic projections.
*   **Differentiation:** It trades image sharpness for significant computational speed and bandwidth compression (ratios of 8:1 to 32:1) by leveraging standard 3D graphics rendering, whereas direct methods are computationally prohibitive for high-resolution full-parallax holograms.
*   **When to choose:** For real-time or high-bandwidth applications where slight blur is acceptable and matched to Human Visual System (HVS) acuity.

### [Holographic Stereogram](7067-holographic-stereogram.md)
*   **What it is:** A hologram composed of discrete 2D perspective views (often Horizontal-Parallax-Only, HPO) of the object scene, similar to lenticular displays but recorded on holographic media.
*   **Differentiation:** It simplifies the 3D problem to a series of 2D holograms, reducing computational complexity compared to full-parallax direct calculation, but limits viewing angles to the parallax axis.
*   **When to choose:** For applications requiring depth perception with limited viewing angles or when computational resources are constrained.

### [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md)
*   **What it is:** Uses an iterative optimization algorithm (ORA) to map desired 3D voxel intensities to the phase-only constraints of an SLM, creating arbitrary 3D focal spot arrays.
*   **Differentiation:** Explicitly handles phase ambiguity through iterative forward-backward propagation and weight updates, unlike direct analytical mapping methods.
*   **When to choose:** When precise control over 3D intensity distributions (e.g., for optical trapping or multi-plane displays) is required using phase-only SLMs.

### [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md)
*   **What it is:** Designs finite-extent holographic samples (e.g., in glass) by determining the optimal 3D distribution of refractive index voxels to convert input modes to target modes.
*   **Differentiation:** Uses an iterative forward-backward propagation scheme maximizing local overlap integrals, rather than simulating diffraction from a surface or optimizing phase on a 2D SLM.
*   **When to choose:** For fabricating static, high-efficiency volume holograms (CGVHs) for specific mode-conversion tasks, such as in optical communications or quantum optics.

### [Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)](10810-multiple-wavefront-recording-planes-wrp-with-horizontal-para.md)
*   **What it is:** Computes holograms from point clouds using multiple parallel wavefront recording planes (WRPs), adapted for HPO by computing contributions to 1D hololines.
*   **Differentiation:** Inherently handles occlusion by using multiple WRPs, avoiding the need for explicit hidden-surface removal algorithms required in single-plane methods.
*   **When to choose:** For point-cloud-based holography where occlusion is significant and full parallax is not required.

### Ray-tracing Method for Full-Parallax CGHs
*   **Definition:** Calculates CGHs by simulating ray paths to remove hidden surfaces and render realistic material properties [S1, unofficial].
*   **Steps:** Cast rays from every elementary hologram into virtual objects; determine traveling paths considering intersection, absorption, reflection, and refraction; express multiple reflections/refractions by casting additional rays; calculate light path length; apply Phong or Cook–Torrance reflection models [S1, unofficial].
*   **Pitfall:** Hidden surfaces must be explicitly removed to avoid incorrect fringe generation [S1, unofficial].

### Multiple Reference Plane (MRP) approach
*   **Definition:** Reduces reconstruction error and blur in holographic stereograms by assigning reference planes to different depth segments of a 3D scene [S1, unofficial].
*   **Steps:** Divide the 3D scene into depth parts; set a reference plane for each part; resynthesize synthetic perspective images referred to their own reference planes; generate the final synthetic perspective image exposed to one holographic elemental by once exposure [S1, unofficial].
*   **Pitfall:** Traditional EPISM causes distortion and blur for object points far from the reference plane [S1, unofficial].

### Holographic Defuser Light Field Display
*   **Definition:** Combines a rotating holographic defuser and a high-speed projector to form multiple viewpoints in 360 degrees [S2, unofficial].
*   **Parameters:** Reflection angle range is 1.25 degrees; the defuser updates the image synchronized with rotation to generate viewpoints [S2, unofficial].

### Color Holovideo via Additive Primaries
*   **Definition:** Produces full-color holographic images by computing three separate fringes for additive primary colors (red, green, blue) accounting for different wavelengths, and modulating three separate laser beams [S3].
*   **Steps:** Compute three separate fringes for red, green, and blue; use each fringe to modulate a separate laser beam corresponding to that primary color [S3].

### Off-axis interferometry for complex field retrieval
*   **Definition:** Measures the complex light field (amplitude and phase) at the output facet of a CGVH by interfering the signal with a tilted reference wave [S4, unofficial].
*   **Steps:** Tilt the reference wave to create interference fringes with a period of ~3 pixels; capture the intensity image; compute FFT; crop a 60x60 pixel region around one sideband; compute inverse FFT to obtain the complex field; apply corrections using a Nelder-Mead optimizer to maximize overlap with the target mode (e.g., HG1,1) [S4, unofficial].
*   **Pitfall:** Retrieved field may show poor overlap due to imperfect centering, geometric rotation, global phase tilt/defocus, or mode waist differences [S4, unofficial].

### Rendering Framework for Autostereoscopic Displays
*   **Steps:** Four pre-processing steps: 1. View extraction (sliding window to acquire sub-holograms); 2. Suppression of aperture diffraction (apodization using 1D Hanning window); 3. Back propagation (zero-padding and BL-ASM); 4. Colour balancing (percentile clipping) [S5].
*   **Complexity:** $O(NM)$, where back propagation requires only 1D FFTs with pre-computed kernels [S5].
*   **Performance:** On a 3.3 GHz Intel Core i7-5820 CPU, processing the 'Dice' dataset (142,903 points) took: View extraction 0.04s, Apodization 0.38s, Back propagation 16.33s, Colour balancing 1.19s [S5].

### Back Propagation
*   **Steps:** The apodized aperture is zero-padded to the full hologram size; back-propagated using BL-ASM to reconstruct the view; the magnitude (square root of intensity) is extracted to match display gamma curves [S5].

## Background
Computer-Generated Holography (CGH) originated from the work of Dennis Gabor, who invented holography in 1948, and Emmett Leith and Juris Upatnieks, who introduced off-axis holography and computer-generated holograms in the early 1960s [S1, unofficial]. The development of CGH was driven by the need to create holograms for objects that were difficult to photograph or did not exist yet. Early methods relied on direct calculation of interference patterns between object and reference waves [S1, unofficial].

The advent of high-performance computing and Spatial Light Modulators (SLMs) in the 1990s and 2000s enabled real-time or near-real-time CGH generation. Key developments include the introduction of the Hogel-based approach for compression [S1, unofficial], iterative phase retrieval algorithms like Gerchberg-Saxton for phase-only SLMs [S2, unofficial], and volume rendering techniques for volumetric data [S3]. Recent advancements focus on optimizing computational efficiency (e.g., elemental holograms, parallel processing) and improving image quality (e.g., multiple reference planes, inverse design for volume holograms) [S1, unofficial][S5].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
[S3] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S4] Design and manufacturing of efficient computer-generated volume ..., https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
[S5] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
