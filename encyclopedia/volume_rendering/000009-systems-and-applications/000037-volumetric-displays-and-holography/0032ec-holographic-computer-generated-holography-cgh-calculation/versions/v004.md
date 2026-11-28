# Holographic Computer-Generated Holography (CGH) Calculation

Also called: CGH Generation, Hologram Synthesis, Computer-Generated Holography

## Overview
Holographic Computer-Generated Holography (CGH) Calculation refers to the computational methods used to synthesize the interference or diffraction patterns required to reconstruct 3D volumetric scenes on spatial light modulators (SLMs) or photographic plates. It solves the problem of mapping 3D scene data (voxels, point clouds, or meshes) into 2D complex amplitude or phase distributions that, when illuminated, produce the correct wavefront for human perception or optical processing. What sets this family apart from sibling methods like [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md) or [Photophoretic-Trap Volumetric Display](6753-photophoretic-trap-volumetric-display.md) is that CGH does not directly emit light from physical points in space; instead, it calculates the *entire* optical field (including interference and diffraction effects) that, when displayed on a 2D plane, recreates the 3D light field through diffraction. This allows for full parallax, occlusion, and depth cues without mechanical scanning of the final image plane, though at the cost of significant computational complexity [S1, unofficial][S3].

## Shared principles
All CGH calculation methods share the fundamental goal of determining the complex optical field $U(x,y)$ at a recording plane (the hologram) such that its propagation to a reconstruction plane yields the desired 3D intensity distribution. The core mathematical framework relies on scalar diffraction theory.

### Wave Propagation
The propagation of light from the object plane to the hologram plane (or between intermediate planes) is modeled using either the **Angular Spectrum Method (ASM)** or the **Fresnel/Kirchhoff diffraction integrals**.
For the Angular Spectrum Method, the field $U(x,y,z)$ at distance $z$ is obtained by:
$$ U(x,y,z) = \mathcal{F}^{-1} \left\{ \mathcal{F}\{U(x,y,0)\} \cdot H(u,v,z) \right\} $$
where $\mathcal{F}$ denotes the Fourier Transform, and $H(u,v,z) = \exp(i 2\pi z \sqrt{\lambda^{-2} - u^2 - v^2})$ is the transfer function in the spatial frequency domain $(u,v)$ [S1, unofficial][S5].

### Point Source Approximation
Many methods (e.g., [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md), [Multiple Wavefront Recording Planes](10810-multiple-wavefront-recording-planes-wrp-with-horizontal-para.md)) treat volumetric elements (voxels or points) as point sources. The complex amplitude contribution of a point source $i$ at position $\mathbf{r}_i$ to a hologram pixel at $\mathbf{r}_h$ is:
$$ U_i(\mathbf{r}_h) = \frac{A_i}{|\mathbf{r}_h - \mathbf{r}_i|} \exp\left(i k |\mathbf{r}_h - \mathbf{r}_i|\right) $$
where $A_i$ is the amplitude (derived from color/opacity maps or point cloud data) and $k = 2\pi/\lambda$ is the wavenumber [S3][S1, unofficial]. The total hologram field is the superposition of all such contributions:
$$ U_{holo} = \sum_{i=1}^{M} U_i $$

### Phase-Only Modulation
Since most SLMs are phase-only, the calculated complex field $U_{holo} = A e^{i\phi}$ must be encoded into a phase pattern $\phi_h$. Common techniques include:
1.  **Detour Phase:** Encoding amplitude and phase into the spatial position of sub-pixels [S1, unofficial].
2.  **Iterative Optimization:** Using algorithms like Gerchberg-Saxton or ORA to find a phase-only hologram that minimizes the error between the reconstructed intensity and the target intensity [S1, S2, unofficial].
3.  **Direct Quantization:** Simply taking the phase component $\phi = \arg(U_{holo})$, which discards amplitude information and introduces noise [S1, unofficial].

### Computational Bottlenecks
The primary challenge is the $O(N^2 \log N)$ or $O(N^3)$ complexity of direct summation for $N$ pixels and $M$ voxels. Strategies to mitigate this include:
*   **Subsampling:** Using hogels (blocks of pixels) to represent local wavefronts [S1, unofficial].
*   **Parallelization:** Leveraging GPUs for FFTs and parallel voxel processing [S1, unofficial][S5].
*   **Approximation:** Using elemental holograms or single-view approximations [S3][S1, unofficial].

## Subtopics

### [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)
This method generates CGHs for volumetric data by treating voxels as transparent objects with diffuse reflection, enabling the reconstruction of transparent and multi-layer objects. It differs from Maximum Intensity Projection (MIP)-based methods, which only display objects with maximum voxel values, by simulating light propagation and attenuation to preserve depth and transparency cues. The approximate variant reduces computational cost by dividing the hologram into elemental holograms and calculating final light propagation only once per elemental hologram using its center pixel.

### [Diffraction-Specific Approach](7015-diffraction-specific-approach.md)
The Diffraction-Specific Approach is a method for calculating Computer-Generated Holograms (CGH) that avoids the direct simulation of optical interference by working backwards from the 3D image data. It subsamples the hologram spatially into functional elements called "hogels" and spectrally into "hogel vectors," enabling significant bandwidth compression and faster computation compared to conventional interference-based methods. Unlike the Interference-Based Approach, which sums complex wavefronts for every pixel, this method leverages standard 3D graphics rendering and precomputed basis fringes to reconstruct the hologram, trading a controlled amount of image sharpness (blur) for substantial gains in speed and data volume.

### [Holographic Stereogram](7067-holographic-stereogram.md)
A Holographic Stereogram is a type of computer-generated hologram composed of a series of discrete 2D perspective views of an object scene. It produces a view-dependent image that presents the corresponding perspective view in each horizontally displaced direction, functioning similarly to a lenticular or parallax barrier display. Unlike full-parallax methods, it restricts vertical parallax, significantly reducing computational complexity while maintaining horizontal depth cues.

### [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md)
Computational Phase Modulation (CPM), specifically implemented via the Optimization with Random Phase Algorithm (ORA), is an iterative optimization method for generating Computer-Generated Holograms (CGHs) that produce a spot array with uniform intensity on a reconstruction plane. It solves the problem of controlling the spatial intensity distribution of light reflected from a Liquid Crystal Spatial Light Modulator (LC-SLM) to create multiple, precisely positioned 3D focal points (voxels) for volumetric displays. Unlike direct superposition methods, ORA iteratively adjusts the phase of the hologram to minimize the error between the reconstructed intensity and a desired target intensity, allowing for high-quality uniform spot arrays.

### [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md)
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing volumetric optical elements that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping. It solves the problem of determining the optimal 3D refractive index distribution within a finite glass sample to achieve high-efficiency mode conversion. Unlike direct forward calculation methods, this algorithm uses an iterative optimization loop that alternates between forward propagation of input modes and backpropagation of target modes to locally maximize an overlap criterion at each voxel layer. It is distinct from sibling methods like [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by focusing on discrete voxel selection from a predefined library of transverse index profiles rather than continuous phase modulation or direct diffraction summation.

### [Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)](10810-multiple-wavefront-recording-planes-wrp-with-horizontal-para.md)
This method generates computer-generated holograms (CGH) for point clouds by computing wavefield propagation across multiple parallel planes, adapted for horizontal parallax only (HPO) displays. It solves the computational bottleneck of full-parallax CGH by restricting diffraction to 1D hololines, while handling occlusion and diffuse reflections through Gaussian volume attribution and inverted filters. It is distinguished from full-parallax WRP methods by its 1D propagation geometry and from single-plane methods by its ability to handle deep scenes with occlusion via intermediate recording planes.

### Ray-tracing Method for Full-Parallax CGHs
#### definition
- [S1, unofficial] Calculates computer-generated holograms by simulating ray paths to remove hidden surfaces and render realistic material properties.
#### step
- [S1, unofficial] Cast rays from every elementary hologram into virtual objects.
- [S1, unofficial] Determine the traveling paths of the rays.
- [S1, unofficial] Consider intersection with objects, absorption, reflection, and refraction during path calculation.
- [S1, unofficial] Express multiple reflections and refractions by casting additional rays into the reflection and transmission directions.
- [S1, unofficial] Calculate the length of the light path for each ray.
- [S1, unofficial] Apply the Phong reflection model and Cook–Torrance reflection model to express material quality.
#### pitfall
- [S1, unofficial] Hidden surfaces must be explicitly removed to avoid incorrect holographic fringe generation.

### Multiple Reference Plane (MRP) approach
#### definition
- [S1, unofficial] Reduces reconstruction error and blur in holographic stereograms by assigning reference planes to different depth segments of a 3D scene.
#### step
- [S1, unofficial] Divide the 3D scene into several parts along its depth direction.
- [S1, unofficial] Set a reference plane for each of the object parts.
- [S1, unofficial] Resynthesize all effectively synthetic perspective images referred to their own reference planes of the object parts.
- [S1, unofficial] Generate the finally effectively synthetic perspective image exposed to one holographic elemental by only once exposure.
#### pitfall
- [S1, unofficial] Traditional EPISM causes distortion and blur for object points located far away from the reference plane.

### Holographic Defuser Light Field Display
#### definition
- [S2, unofficial] A system combining a rotating holographic defuser and a high-speed projector to form multiple viewpoints in 360 degrees. The holographic defuser reflects images within a limited range of 1.25 degrees, updating the image synchronized with rotation to generate viewpoints.
#### parameter
- [S2, unofficial] Reflection angle range: 1.25 degrees.

### Color Holovideo via Additive Primaries
#### definition
- [S3] Production of full-color holographic images by computing three separate fringes for additive primary colors (red, green, blue) accounting for different wavelengths, and modulating three separate laser beams.
#### step
- [S3] 1. Compute three separate fringes, one for each additive primary color (red, green, blue), considering their specific wavelengths. 2. Use each fringe to modulate a separate beam of light corresponding to that primary color.

### Off-axis interferometry for complex field retrieval
#### definition
- [S4, unofficial] A technique to measure the complex light field (amplitude and phase) at the output facet of a CGVH by interfering the signal with a tilted reference wave, allowing characterization of mode sorting quality.
#### step
- [S4, unofficial] Tilt the reference wave such that interference fringes have a period of about 3 pixels.
- [S4, unofficial] Capture the intensity image (interferogram).
- [S4, unofficial] Compute the Fast Fourier Transform (FFT) of the intensity image.
- [S4, unofficial] Crop a 60x60 pixel region around one sideband in the frequency domain.
- [S4, unofficial] Compute the inverse FFT of the cropped sideband to obtain the complex field.
- [S4, unofficial] Apply corrections using a Nelder-Mead optimizer to maximize the overlap integral between the measured field (at 0° AOI) and the target mode (e.g., HG1,1). The seven optimized parameters are then used to correct all experimentally recorded fields.
#### pitfall
- [S4, unofficial] The retrieved field may show poor overlap with target modes due to: 1) imperfect centering on the pixel grid, 2) small geometric rotation in experimental images, 3) global phase tilt and defocus, and 4) slight differences in mode waist compared to simulation.

### Rendering Framework for Autostereoscopic Displays
#### step
- [S5] The framework consists of four sequential pre-processing steps applied to the hologram to acquire a set of reconstruction images corresponding to views from a camera rotating around the scene. The resolution of these images equals the 2D resolution of the screen.
- [S5] Step 1: View extraction. Step 2: Suppression of aperture diffraction (apodization). Step 3: Back propagation. Step 4: Colour balancing.
#### complexity
- [S5] The computational complexity of the framework is $O(NM)$. The back propagation requires only 1D FFTs with a pre-computed kernel due to the same distance.
#### number
- [S5] Performance test on a 3.3 GHz Intel Core i7-5820 CPU with Windows 8 (x64). For the 'Dice' dataset (142,903 points), processing times per view were: View extraction 0.04s, Apodization 0.38s, Back propagation 16.33s, Colour balancing 1.19s.
- [S5] Rendering on an Holografika HoloVizio 722RC light field display (72 projectors, 70° viewing angle). 70 views were extracted with an angular resolution of 1° and a 2D resolution of 1280x768 pixels.

### Back Propagation
#### step
- [S5] The apodized aperture is zero-padded to the size of the full hologram to represent the hologram from that specific viewing position. It is then back-propagated using BL-ASM to reconstruct the hologram for the corresponding view.
- [S5] To visualize the reconstructed object, the magnitude (square root of the intensity) of the complex wavefield computed by the back propagation is extracted. This magnitude matches the gamma curves for displays and is stored as a 2D image.

## Background
The concept of Computer-Generated Holography (CGH) originated from the work of Emmett Leith and Juris Upatnieks in 1962, who demonstrated that holograms could be synthesized mathematically rather than through optical interference [S1, unofficial]. Early CGH methods were computationally prohibitive due to the need to calculate interference patterns for every point in a scene. The advent of powerful GPUs and parallel computing has enabled the practical application of CGH for volumetric displays and holographic light field rendering.

Key developments include:
*   **Holographic Stereograms:** Early methods that used perspective images to reduce computational load, though limited in parallax [S1, unofficial].
*   **Point Cloud CGH:** Treating 3D scenes as collections of point sources, allowing for efficient superposition of spherical waves [S3].
*   **Hogel-based Methods:** Subsampling techniques like the Diffraction-Specific Approach to compress data and speed up rendering [S1, unofficial].
*   **Iterative Phase Retrieval:** Algorithms like Gerchberg-Saxton and ORA to handle phase-only SLMs [S1, unofficial].
*   **Inverse Design:** Recent advances using optimization to design volumetric optical elements (CGVHs) for specific mode conversions [S1, unofficial].

CGH is used in volumetric displays, holographic light field displays, optical data storage, and mode sorting in fiber optics [S1, S4, unofficial][S5].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
[S3] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S4] Design and manufacturing of efficient computer-generated volume ..., https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
[S5] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
