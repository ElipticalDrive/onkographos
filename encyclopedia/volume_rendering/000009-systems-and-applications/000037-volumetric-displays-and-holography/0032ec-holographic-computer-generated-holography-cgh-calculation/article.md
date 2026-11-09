# Holographic Computer-Generated Holography (CGH) Calculation

Also called: Computer-Generated Hologram, CGVH (Computer-Generated Volume Hologram)

## Overview
Holographic Computer-Generated Holography (CGH) calculation is the computational process of synthesizing the complex optical field (amplitude and phase) required to reconstruct a 3D scene on a Spatial Light Modulator (SLM) or photographic plate. It solves the problem of translating discrete volumetric data (voxels, point clouds, or meshes) into a continuous interference pattern that, when illuminated, produces a light field matching the target 3D geometry. What sets CGH apart from standard volume rendering (e.g., ray casting) is that it explicitly simulates wave optics—interference and diffraction—rather than geometric ray transport, enabling the reconstruction of transparent, occluded, and multi-layer objects with correct depth cues and parallax [S3].

## Shared principles
All CGH methods share the fundamental goal of determining a hologram function $H(x,y)$ (typically phase-only or complex-valued) such that the diffracted light field $U_{rec}$ at a reconstruction plane approximates a target field $U_{target}$.

The core physical model relies on the superposition of waves. For a point source or voxel $i$ at position $\mathbf{r}_i$ with complex amplitude $A_i$, the contribution to the hologram plane at $\mathbf{r}_h$ is given by the spherical wave propagation:
$$ U_i(\mathbf{r}_h) = \frac{A_i}{|\mathbf{r}_h - \mathbf{r}_i|} \exp\left(i k |\mathbf{r}_h - \mathbf{r}_i|\right) $$
where $k = 2\pi/\lambda$ is the wavenumber. The total hologram field is the sum of contributions from all scene elements:
$$ U_{hologram}(\mathbf{r}_h) = \sum_i U_i(\mathbf{r}_h) $$

Key shared constraints include:
1.  **Sampling:** The hologram pixel pitch $p$ must satisfy the Nyquist criterion for the highest spatial frequency in the object spectrum to avoid aliasing [S1, Parent, unofficial]. The minimum object-hologram distance $z_{min}$ is often constrained by $z_{min} = \frac{N p^2}{\lambda}$ [S1, unofficial].
2.  **Phase Quantization:** SLMs typically support phase-only modulation. Methods must convert complex amplitudes to phase values $\phi_h \in [0, 2\pi)$, often using detour phase, detour phase with carrier frequencies, or iterative optimization [S2, unofficial].
3.  **Diffraction Limits:** The maximum diffraction angle $\theta$ is limited by the pixel pitch: $\sin \theta = \frac{\lambda}{2p}$ [S1, unofficial].

## Subtopics

### [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)
This method treats voxels as transparent objects with diffuse reflection, enabling the reconstruction of transparent and multi-layer objects. It differs from Maximum Intensity Projection (MIP)-based methods by simulating light propagation and attenuation to preserve depth and transparency cues. The approximate variant reduces computational cost by dividing the hologram into elemental holograms and calculating final light propagation only once per elemental hologram using its center pixel.

### [Diffraction-Specific Approach](7015-diffraction-specific-approach.md)
This approach avoids direct optical interference simulation by subsampling the hologram into "hogels" and spectrally into "hogel vectors". It replaces billions of complex arithmetic operations with standard 3D computer graphics rendering (orthographic projections) and efficient decoding via texture mapping. It trades image sharpness for significant bandwidth compression and computation speed, with added blur matched to the acuity of the human visual system (HVS).

### [Holographic Stereogram](7067-holographic-stereogram.md)
A Holographic Stereogram is composed of a series of discrete 2D perspective views of an object scene. It produces a view-dependent image that presents the corresponding perspective view in each horizontally displaced direction, functioning similarly to lenticular or parallax barrier displays. Unlike full-parallax methods, it restricts depth cues to horizontal motion, significantly reducing computational load compared to full 3D wavefront reconstruction.

### [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md)
Computational Phase Modulation, specifically via the Optimization with Random Phase Algorithm (ORA), is an iterative optimization method for generating Computer-Generated Holograms (CGH) that produce a target intensity distribution $I_r$ composed of a uniform spot array. It solves the phase retrieval problem for phase-only Spatial Light Modulators (SLMs) by iteratively adjusting the hologram phase $\phi_h$ to maximize the sum of light intensities at desired reconstruction pixels. It differs from direct diffraction calculation methods by using feedback-driven optimization rather than analytical superposition, and from standard Gerchberg-Saxton implementations by its specific update rule for phase variation $\Delta\phi_h$ derived from maximizing intensity sums.

### [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md)
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing volumetric optical elements that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping. It solves the phase retrieval and structural optimization problem for 3D refractive index distributions by iteratively selecting discrete voxel patterns to maximize the overlap between propagated fields and target modes. Unlike direct diffraction calculation methods which sum contributions analytically, this approach uses an iterative optimization loop (Forward-Backward propagation) to determine the optimal refractive index change $\Delta RI$ for each voxel location. It is distinct from general phase-modulation CGH methods by specifically targeting multi-mode conversion in a volume medium using a split-step Fourier Beam Propagation Method (BPM).

### [Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)](10810-multiple-wavefront-recording-planes-wrp-with-horizontal-para.md)
This method generates computer-generated holograms (CGH) for point clouds by computing contributions to multiple parallel wavefront recording planes (WRPs), adapted for horizontal parallax only (HPO) displays. It solves the high computational cost of full-parallax CGH by restricting diffraction to 1D hololines, while using the WRP strategy to handle occlusion and enable fast propagation via look-up tables (LUTs). It is distinguished from full-parallax WRP methods by its 1D computation per point and from direct diffraction methods by its use of parallel propagation planes and Gaussian point volumes.

### Ray-tracing Method for Full-Parallax CGHs
- **Definition:** Calculates computer-generated holograms by simulating ray paths to remove hidden surfaces and render realistic material properties [S1, unofficial].
- **Steps:** Cast rays from every elementary hologram into virtual objects. Determine the traveling paths of the rays. Consider intersection with objects, absorption, reflection, and refraction during path calculation. Express multiple reflections and refractions by casting additional rays into the reflection and transmission directions. Calculate the length of the light path for each ray. Apply the Phong reflection model and Cook–Torrance reflection model to express material quality [S1, unofficial].
- **Pitfall:** Hidden surfaces must be explicitly removed to avoid incorrect holographic fringe generation [S1, unofficial].

### Multiple Reference Plane (MRP) approach
- **Definition:** Reduces reconstruction error and blur in holographic stereograms by assigning reference planes to different depth segments of a 3D scene [S1, unofficial].
- **Steps:** Divide the 3D scene into several parts along its depth direction. Set a reference plane for each of the object parts. Resynthesize all effectively synthetic perspective images referred to their own reference planes of the object parts. Generate the finally effectively synthetic perspective image exposed to one holographic elemental by only once exposure [S1, unofficial].
- **Pitfall:** Traditional EPISM causes distortion and blur for object points located far away from the reference plane [S1, unofficial].

### Holographic Defuser Light Field Display
- **Definition:** A system combining a rotating holographic defuser and a high-speed projector to form multiple viewpoints in 360 degrees. The holographic defuser reflects images within a limited range of 1.25 degrees, updating the image synchronized with rotation to generate viewpoints [S2, unofficial].
- **Parameter:** Reflection angle range: 1.25 degrees [S2, unofficial].

### Color Holovideo via Additive Primaries
- **Definition:** Production of full-color holographic images by computing three separate fringes for additive primary colors (red, green, blue) accounting for different wavelengths, and modulating three separate laser beams [S3].
- **Steps:** 1. Compute three separate fringes, one for each additive primary color (red, green, blue), considering their specific wavelengths. 2. Use each fringe to modulate a separate beam of light corresponding to that primary color [S3].

### Off-axis interferometry for complex field retrieval
- **Definition:** A technique to measure the complex light field (amplitude and phase) at the output facet of a CGVH by interfering the signal with a tilted reference wave, allowing characterization of mode sorting quality [S4, unofficial].
- **Steps:** Tilt the reference wave such that interference fringes have a period of about 3 pixels. Capture the intensity image (interferogram). Compute the Fast Fourier Transform (FFT) of the intensity image. Crop a 60x60 pixel region around one sideband in the frequency domain. Compute the inverse FFT of the cropped sideband to obtain the complex field. Apply corrections using a Nelder-Mead optimizer to maximize the overlap integral between the measured field (at 0° AOI) and the target mode (e.g., HG1,1). The seven optimized parameters are then used to correct all experimentally recorded fields [S4, unofficial].
- **Pitfall:** The retrieved field may show poor overlap with target modes due to: 1) imperfect centering on the pixel grid, 2) small geometric rotation in experimental images, 3) global phase tilt and defocus, and 4) slight differences in mode waist compared to simulation [S4, unofficial].

### Rendering Framework for Autostereoscopic Displays
- **Steps:** The framework consists of four sequential pre-processing steps applied to the hologram to acquire a set of reconstruction images corresponding to views from a camera rotating around the scene. The resolution of these images equals the 2D resolution of the screen [S5]. Step 1: View extraction. Step 2: Suppression of aperture diffraction (apodization). Step 3: Back propagation. Step 4: Colour balancing [S5].
- **Complexity:** The computational complexity of the framework is $O(NM)$. The back propagation requires only 1D FFTs with a pre-computed kernel due to the same distance [S5].
- **Number:** Performance test on a 3.3 GHz Intel Core i7-5820 CPU with Windows 8 (x64). For the 'Dice' dataset (142,903 points), processing times per view were: View extraction 0.04s, Apodization 0.38s, Back propagation 16.33s, Colour balancing 1.19s [S5]. Rendering on an Holografika HoloVizio 722RC light field display (72 projectors, 70° viewing angle). 70 views were extracted with an angular resolution of 1° and a 2D resolution of 1280x768 pixels [S5].

### Back Propagation
- **Steps:** The apodized aperture is zero-padded to the size of the full hologram to represent the hologram from that specific viewing position. It is then back-propagated using BL-ASM to reconstruct the hologram for the corresponding view [S5]. To visualize the reconstructed object, the magnitude (square root of the intensity) of the complex wavefield computed by the back propagation is extracted. This magnitude matches the gamma curves for displays and is stored as a 2D image [S5].

## Background
Computer-generated holography was pioneered by Emmett Leith and Juris Upatnieks in 1962, who demonstrated the recording of holograms using computer-generated interference patterns [S1, unofficial]. Early methods focused on point-cloud representations. As computational power increased, methods evolved to handle complex volumetric data. The integration of CGH with volume rendering techniques allows for the visualization of medical and scientific data in 3D [S1, unofficial]. Recent developments include inverse design algorithms for volumetric optical elements (CGVHs) fabricated in glass, enabling efficient mode sorting and complex wavefront shaping [S4, unofficial]. Additionally, hybrid systems combining CGH with light field displays and scanning mechanisms have been developed to achieve large viewing angles and real-time updates [S2, unofficial][S5].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
[S3] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S4] Design and manufacturing of efficient computer-generated volume ..., https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
[S5] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
