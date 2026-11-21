# Holographic Computer-Generated Holography (CGH) Calculation

Also called: Computer-Generated Hologram, CGVH (Computer-Generated Volume Hologram)

## Overview
Holographic Computer-Generated Holography (CGH) Calculation is the computational process of synthesizing the complex optical field (amplitude and phase) required to reconstruct a 3D scene on a display device, such as a Spatial Light Modulator (SLM) or printed film [S1, unofficial][S3]. It solves the problem of converting discrete volumetric data or point clouds into interference patterns that, when illuminated, produce the correct wavefront for human perception [S1, unofficial]. Unlike standard volume rendering which projects 2D images, CGH explicitly models diffraction and interference to preserve depth cues, parallax, and focus accommodation [S3]. The family is distinguished by its trade-off between optical fidelity and computational complexity, ranging from direct point-source summation to compressed spectral representations and iterative phase optimization [S1, unofficial][S3].

## Shared principles
All methods in this family share the goal of generating a complex field $U(x,y)$ or phase-only pattern $\phi(x,y)$ on a 2D plane (the hologram plane) that, upon propagation, reconstructs a target 3D intensity distribution $I_r$ [S1, unofficial][S3].

**Common Mathematical Foundations:**
1.  **Wave Propagation:** The core operation is propagating light from object points to the hologram plane. This is typically modeled using the Fresnel or Angular Spectrum Method (ASM) [S1, unofficial][S5]. The complex amplitude $U_h$ at the hologram plane from a point source at $\mathbf{r}_i$ is:
    $$ U_i(\xi, \eta) = A_i \frac{e^{ik r_i}}{r_i} $$
    where $A_i$ is the source amplitude, $k = 2\pi/\lambda$ is the wavenumber, and $r_i$ is the distance from the source to the hologram pixel $(\xi, \eta)$ [S3].
2.  **Superposition:** The total hologram field is the linear superposition of contributions from all object points or voxels:
    $$ U_{holo}(\xi, \eta) = \sum_{i=1}^{M} U_i(\xi, \eta) $$
    where $M$ is the number of object elements [S3].
3.  **Encoding:** Since most display devices (LCOS-SLMs) are phase-only or amplitude-only, the complex field must be encoded. Common techniques include direct phase quantization ($\phi = \arg(U)$), detour phase, or iterative algorithms like Gerchberg-Saxton or ORA to optimize phase for target intensity [S1, unofficial][S3].
4.  **Sampling Constraints:** The hologram resolution and pixel pitch must satisfy the Nyquist criterion for the highest spatial frequency present in the interference pattern, which depends on the viewing angle and wavelength [S1, unofficial][S5].

## Subtopics

### [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)
This method treats voxels as transparent objects with diffuse reflection, simulating light propagation and attenuation to preserve depth and transparency cues. It differs from Maximum Intensity Projection (MIP) by calculating incident and reflected light intensities using a Lambertian model, accounting for occlusion by preceding and subsequent voxels. The approximate variant reduces cost by dividing the hologram into elemental holograms and calculating propagation only once per element using its center pixel.

### [Diffraction-Specific Approach](7015-diffraction-specific-approach.md)
Also known as the Hogel Vector Approach, this method avoids direct wavefront simulation by treating the hologram as a subsampled array of "hogels" represented by spectral vectors. It builds on standard 3D graphics rendering (orthographic projections) to generate these vectors, enabling bandwidth compression and faster computation compared to interference-based methods. Unlike volume rendering CGH, it decouples scene generation from fringe calculation using precomputed basis fringes.

### [Holographic Stereogram](7067-holographic-stereogram.md)
A Holographic Stereogram generates a view-dependent image by composing discrete 2D perspective views, trading vertical parallax for computational efficiency. It functions similarly to lenticular displays, mapping each view to a horizontal segment in the hologram plane. This allows for nearly interactive rates on specialized hardware but requires sufficient view density to avoid sampling artifacts.

### [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md)
This iterative method, specifically using the Optimization with Random Phase Algorithm (ORA), generates phase-only CGHs for target intensity distributions (e.g., spot arrays). It solves the problem of encoding complex fields into phase-only SLMs by optimizing phase to minimize reconstruction error, unlike direct summation which discards amplitude information. It operates on global optimization of the hologram plane rather than direct point-source contributions.

### [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md)
This algorithm designs 3D refractive index structures (voxels) to convert input optical modes to target modes with high efficiency. It uses an iterative forward-backward propagation scheme to select optimal voxel types at each layer, differing from direct phase profile calculations. It builds on [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md) by focusing on discrete voxel optimization for mode sorting.

### [Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)](10810-multiple-wavefront-recording-planes-wrp-with-horizontal-para.md)
This method generates CGHs for point clouds by propagating wavefields across multiple parallel Wavefront Recording Planes (WRPs), restricted to Horizontal Parallax Only (HPO) to reduce complexity. It uses pre-computed Look-Up Tables (LUTs) and the Band-Limited Angular Spectrum Method (BL-ASM) to handle high-bandwidth sampling without aliasing. It distinguishes itself from full-parallax methods by processing 1D hololines and from direct summation by using pre-computed kernels.

### Ray-tracing Method for Full-Parallax CGHs
- **Definition:** Calculates CGHs by simulating ray paths to remove hidden surfaces and render realistic material properties [S1, unofficial].
- **Steps:** Cast rays from every elementary hologram into virtual objects; determine traveling paths considering intersection, absorption, reflection, and refraction; express multiple reflections/refractions by casting additional rays; calculate light path length; apply Phong and Cook–Torrance models for material quality [S1, unofficial].
- **Pitfall:** Hidden surfaces must be explicitly removed to avoid incorrect fringe generation [S1, unofficial].

### Multiple Reference Plane (MRP) approach
- **Definition:** Reduces reconstruction error and blur in holographic stereograms by assigning reference planes to different depth segments of a 3D scene [S1, unofficial].
- **Steps:** Divide the 3D scene into depth segments; set a reference plane for each part; resynthesize synthetic perspective images referred to their own reference planes; generate the final synthetic image exposed to one holographic elemental by once exposure [S1, unofficial].
- **Pitfall:** Traditional EPISM causes distortion and blur for points far from the reference plane [S1, unofficial].

### Holographic Defuser Light Field Display
- **Definition:** Combines a rotating holographic defuser and a high-speed projector to form 360-degree viewpoints [S2, unofficial].
- **Parameter:** Reflection angle range is 1.25 degrees [S2, unofficial].

### Color Holovideo via Additive Primaries
- **Definition:** Produces full-color holograms by computing three separate fringes for red, green, and blue, accounting for different wavelengths [S3].
- **Steps:** Compute three separate fringes for additive primaries; modulate three separate laser beams with each fringe [S3].

### Off-axis interferometry for complex field retrieval
- **Definition:** Measures the complex light field at the output of a CGVH by interfering the signal with a tilted reference wave [S4, unofficial].
- **Steps:** Tilt reference wave for ~3-pixel fringe period; capture intensity image; FFT the image; crop 60x60 pixel region around one sideband; inverse FFT to obtain complex field; apply Nelder-Mead optimization to correct for centering, rotation, phase tilt, defocus, and waist differences [S4, unofficial].
- **Pitfall:** Poor overlap with target modes due to imperfect grid centering, geometric rotation, phase tilt/defocus, and waist differences [S4, unofficial].

### Rendering Framework for Autostereoscopic Displays
- **Steps:** View extraction; suppression of aperture diffraction (apodization); back propagation; colour balancing [S5].
- **Complexity:** $O(NM)$, where back propagation uses 1D FFTs with pre-computed kernel [S5].
- **Performance:** On a 3.3 GHz Intel Core i7-5820 CPU, processing times for the 'Dice' dataset (142,903 points) per view were: View extraction 0.04s, Apodization 0.38s, Back propagation 16.33s, Colour balancing 1.19s [S5].

### Back Propagation
- **Steps:** Zero-pad the apodized aperture to hologram size; back-propagate using BL-ASM to reconstruct the view; extract magnitude (square root of intensity) matching display gamma curves [S5].

## Background
The concept of Computer-Generated Holography was introduced by Leith and Upatnieks in 1962, who described the hologram as a communication channel for wavefronts [S1, unofficial]. Early methods focused on point-source summation, which became computationally prohibitive for large scenes. The development of volumetric displays and holographic light fields in the 2000s and 2010s spurred new CGH algorithms, including ray-tracing for realism [S1, unofficial], hogel-vector compression [S1, unofficial], and iterative phase optimization [S1, unofficial]. Recent advances include inverse design for volume holograms in glass [S4, unofficial] and efficient HPO methods using WRP and LUTs [S1, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
[S3] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S4] Design and manufacturing of efficient computer-generated volume ..., https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
[S5] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
