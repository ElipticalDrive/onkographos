# Holographic Computer-Generated Holography (CGH) Calculation

Also called: Volumetric CGH, Holographic Volume Rendering

## Overview
Holographic Computer-Generated Holography (CGH) Calculation maps 3D voxel data to a physical 3D light field by computing the interference pattern (hologram) required to reconstruct the volume. It differs from sibling methods like [Volumetric Display using Rotating Mirror](2567-volumetric-display-using-rotating-mirror-vrdm.md) or [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md) by using optical superposition (diffraction) rather than temporal persistence or scanning beams to create the 3D illusion. It builds on [Volumetric displays and holography](51-volumetric-displays-and-holography.md) by explicitly modeling wave optics to resolve the viewing space vs. drawing space conflict through phase modulation.

## Shared principles
All methods in this family share the goal of mapping 3D voxel data $I(x,y,z)$ to a physical 3D light field. The core challenge is resolving the "viewing space" vs. "drawing space" conflict: the medium used to generate voxels (e.g., gas, laser focus, holographic plane) often differs from the space where the user observes them. Common mechanisms include:
1.  **Spatial Mapping:** Transforming 3D voxel coordinates $(x,y,z)$ into 2D control signals (pixel buffers, SLM patterns, or scanner coordinates) synchronized with mechanical motion (rotation, scanning) or optical projection.
2.  **Light Propagation Modeling:** Calculating how light interacts with voxels, including attenuation, reflection, and diffraction, to determine the intensity and phase required at the output plane.
3.  **Temporal/Spatial Multiplexing:** Using persistence of vision (rotating mirrors, scanning beams) or optical superposition (holography) to create the illusion of a continuous 3D volume from discrete 2D slices or points.

## Subtopics

### [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)
This method generates Computer-Generated Holograms (CGHs) for volumetric data by treating voxels as transparent objects with diffuse reflection, enabling the reconstruction of transparent and multi-layer objects. It differs from Maximum Intensity Projection (MIP)-based methods, which only display objects with maximum voxel values, by simulating light propagation and attenuation to preserve depth and transparency cues. The approximate variant reduces computational cost by dividing the hologram into elemental holograms and calculating final light propagation only once per elemental hologram using its center pixel.

### [Diffraction-Specific Approach](7015-diffraction-specific-approach.md)
An approach that breaks from traditional simulation of optical holographic interference by working backwards from the 3D image. The fringe is treated as being subsampled spatially into functional holographic elements ('hogels') and spectrally into an array of 'hogel vectors'.

### [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md)
Uses a Liquid Crystal SLM (LCSLM) to modulate the phase of a reflected laser beam. The SLM acts as an optical phased array where molecule directions are controlled by electrodes (pixels) to modulate phase.

### Ray-tracing Method for Full-Parallax CGHs
- **Definition**: Calculates computer-generated holograms by simulating ray paths to remove hidden surfaces and render realistic material properties [S1, unofficial].
- **Steps**:
    - Cast rays from every elementary hologram into virtual objects [S1, unofficial].
    - Determine the traveling paths of the rays [S1, unofficial].
    - Consider intersection with objects, absorption, reflection, and refraction during path calculation [S1, unofficial].
    - Express multiple reflections and refractions by casting additional rays into the reflection and transmission directions [S1, unofficial].
    - Calculate the length of the light path for each ray [S1, unofficial].
    - Apply the Phong reflection model and Cook–Torrance reflection model to express material quality [S1, unofficial].
- **Pitfalls**: Hidden surfaces must be explicitly removed to avoid incorrect holographic fringe generation [S1, unofficial].

### Multiple Reference Plane (MRP) approach
- **Definition**: Reduces reconstruction error and blur in holographic stereograms by assigning reference planes to different depth segments of a 3D scene [S1, unofficial].
- **Steps**:
    - Divide the 3D scene into several parts along its depth direction [S1, unofficial].
    - Set a reference plane for each of the object parts [S1, unofficial].
    - Resynthesize all effectively synthetic perspective images referred to their own reference planes of the object parts [S1, unofficial].
    - Generate the finally effectively synthetic perspective image exposed to one holographic elemental by only once exposure [S1, unofficial].
- **Pitfalls**: Traditional EPISM causes distortion and blur for object points located far away from the reference plane [S1, unofficial].

### Scanned AOM Holographic Display
- **Definition**: A time-multiplexed holographic display technique that uses a fast Acousto-Optic Modulator (AOM) to phase-modulate a small number of samples (~1000) at any instant, scanning the modulated light beam rapidly to create a larger apparent holographic fringe integrated by the human visual system [S2].
- **Parameters**:
    - Human visual system latency: typically 20 ms. The eye time-integrates the displayed fringe during this interval [S2].
    - AOM instantaneous capacity: phase modulates about 1000 samples at any one instant in time [S2].
- **Pitfalls**:
    - Need to convert digitally computed fringes into high-frequency analog signals. This required a specialized 16-channel synchronized high-speed framebuffer system at MIT, which was a major practical obstacle [S2].
    - Requires state-of-the-art scanning mirrors synchronized to the fringe data stream, adding complexity compared to LCD-based displays [S2].
- **Numbers**:
    - First images produced were 50 ml volume, generated from 2 MB fringes [S2].
    - Scaled system with 18 parallel modulation channels produced images >1 liter volume from 36 MB fringes [S2].
- **History**: Invented and exploited by researchers at the MIT Media Laboratory to produce the world's first real-time 3D holographic display in 1989 [S2].

### Holographic Stereogram
- **Definition**: A type of hologram composed of a series of discrete 2D perspective views of the object scene. An HPO (Horizontal-Parallax-Only) stereogram produces a view-dependent image presenting the corresponding perspective view in each horizontally displaced direction, similar to lenticular or parallax barrier displays [S2].
- **Steps**:
    - Generate a sequence of view images by moving the camera laterally in steps [S2].
    - Combine these images to generate a fringe for display [S2].
- **Pitfalls**: Requires a large number of perspective views to create a high-quality image free from sampling artifacts, which limits computation speed [S2].
- **Numbers**: Allows computation at nearly interactive rates when implemented on specialized hardware [S2].

### Holographic Defuser Light Field Display
- **Definition**: A system combining a rotating holographic defuser and a high-speed projector to form multiple viewpoints in 360 degrees. The holographic defuser reflects images within a limited range of 1.25 degrees, updating the image synchronized with rotation to generate viewpoints [S3, unofficial].
- **Parameters**: Reflection angle range: 1.25 degrees [S3, unofficial].

### Color Holovideo via Additive Primaries
- **Definition**: Production of full-color holographic images by computing three separate fringes for additive primary colors (red, green, blue) accounting for different wavelengths, and modulating three separate laser beams [S2].
- **Steps**:
    - Compute three separate fringes, one for each additive primary color (red, green, blue), considering their specific wavelengths [S2].
    - Use each fringe to modulate a separate beam of light corresponding to that primary color [S2].

## Background
The concept of holography was introduced by Dennis Gabor in 1948, with the two-step imaging process described from a communication-theory viewpoint by Leith and Upatnieks in 1962 [S1, unofficial]. Early volumetric display concepts using scanning beams and reflective screens were pioneered by Parker and Wallis in 1948, and Withey in 1958 using fluorescent screens in a vacuum [S3, unofficial]. The first real-time 3D holographic display using a scanned Acousto-Optic Modulator (AOM) was developed by researchers at the MIT Media Laboratory in 1989 [S2]. Modern volumetric CGH calculation methods, such as those treating voxels as transparent diffuse reflectors, have been developed to handle complex volumetric data, addressing the high computational cost and resolution limitations of earlier techniques [S1, S3, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S3] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
