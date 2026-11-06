# Volumetric displays and holography

Also called: Volumetric holography, CGH for volume data, VRDM

## Overview
This family covers hardware systems and computational methods that render volumetric data directly into physical space, creating images that occupy a 3D volume visible from multiple viewpoints without headgear. It differs from standard rasterization or ray marching by projecting light into a physical medium (air, glass, or light fields) rather than onto a 2D sensor or screen. It is distinct from [Voxel engines and games](47-voxel-engines-and-games.md) which simulate volume appearance on 2D displays, and from [Scientific and medical visualization](49-scientific-and-medical-visualization.md) which typically uses 2D slice views or volume rendering on monitors.

## Shared principles
All methods in this family share the goal of mapping 3D voxel data $I(x,y,z)$ to a physical 3D light field. The core challenge is resolving the "viewing space" vs. "drawing space" conflict: the medium used to generate voxels (e.g., gas, laser focus, holographic plane) often differs from the space where the user observes them. Common mechanisms include:
1.  **Spatial Mapping:** Transforming 3D voxel coordinates $(x,y,z)$ into 2D control signals (pixel buffers, SLM patterns, or scanner coordinates) synchronized with mechanical motion (rotation, scanning) or optical projection.
2.  **Light Propagation Modeling:** Calculating how light interacts with voxels, including attenuation, reflection, and diffraction, to determine the intensity and phase required at the output plane.
3.  **Temporal/Spatial Multiplexing:** Using persistence of vision (rotating mirrors, scanning beams) or optical superposition (holography) to create the illusion of a continuous 3D volume from discrete 2D slices or points.

## Subtopics

### [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)
Generates Computer-Generated Holograms (CGHs) by treating voxels as transparent diffuse reflectors. It simulates light propagation and attenuation to preserve depth and transparency, unlike MIP-based methods which only show maximum intensity. It is chosen when high-fidelity transparency and multi-layer reconstruction are required, despite high computational cost.

### [Volumetric Display using Rotating Mirror (VRDM)](2567-volumetric-display-using-rotating-mirror-vrdm.md)
Projects 3D voxel data onto surrounding LED screens via a rotating mirror to create a stationary volumetric image. It uses analytical mirror reflection matrices to map 3D coordinates to 2D pixels, avoiding per-pixel ray tracing. It is chosen for hardware systems requiring real-time volumetric projection with occlusion handling via depth sorting.

### [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md)
Renders volumetric images with arbitrary colors per voxel using a beam scanner and an LC color filter. It is chosen for systems requiring full-color volumetric displays with precise spatial control of individual voxels.

### [Drawing Space Separation (DSS)](5290-drawing-space-separation-dss.md)
Separates the "drawing space" (where voxels are generated, e.g., in xenon gas) from the "viewing space" using a re-projection system. This allows the use of materials with low energy thresholds for voxel generation without physical barriers. It is chosen when the display medium requires specific environmental conditions (like gas discharge) that are incompatible with direct user viewing.

### [Dual-Path Holographic Laser Rendering](5627-dual-path-holographic-laser-rendering.md)
Renders palm-sized volumetric contents in real space by splitting a femtosecond laser into two paths: one for holographic voxel generation (LC-SLM) and one for 3D laser scanning. It is chosen for high-resolution, real-time volumetric displays in free space.

### Ray-tracing Method for Full-Parallax CGHs
**Definition:** A method that calculates computer-generated holograms by simulating ray paths to remove hidden surfaces and render realistic material properties [S1, unofficial].
**Steps:**
1.  Cast rays from every elementary hologram into virtual objects [S1, unofficial].
2.  Determine the traveling paths of the rays [S1, unofficial].
3.  Consider intersection with objects, absorption, reflection, and refraction during path calculation [S1, unofficial].
4.  Express multiple reflections and refractions by casting additional rays into the reflection and transmission directions [S1, unofficial].
5.  Calculate the length of the light path for each ray [S1, unofficial].
6.  Apply the Phong reflection model and Cook–Torrance reflection model to express material quality [S1, unofficial].
**Pitfalls:** Hidden surfaces must be explicitly removed to avoid incorrect holographic fringe generation [S1, unofficial].

### Multiple Reference Plane (MRP) approach
**Definition:** Reduces reconstruction error and blur in holographic stereograms by assigning reference planes to different depth segments of a 3D scene [S1, unofficial].
**Steps:**
1.  Divide the 3D scene into several parts along its depth direction [S1, unofficial].
2.  Set a reference plane for each of the object parts [S1, unofficial].
3.  Resynthesize all effectively synthetic perspective images referred to their own reference planes of the object parts [S1, unofficial].
4.  Generate the finally effectively synthetic perspective image exposed to one holographic elemental by only once exposure [S1, unofficial].
**Pitfalls:** Traditional EPISM causes distortion and blur for object points located far away from the reference plane [S1, unofficial].

### Varifocal Mirror Display
**Definition:** A display system that eliminates the need for mental reconstruction by displaying the entire stack of tomographic images as a 3D image using a vibrating mirror with variable focal length optics [S2, unofficial].
**Steps:**
1.  The speaker is made to vibrate at 30 Hz [S2, unofficial].
2.  Tomographic images are displayed one at a time on the CRT in synchrony with the motion of the mirror [S2, unofficial].
3.  Due to the variable focal length of the mirror, the timing sequence of each frame, and retinal integration, the stack of 2D images appear as a virtual 3D image in the mirror [S2, unofficial].
**Layout:** The mirror consists of an aluminized membrane stretched over a loud-speaker coupled to a CRT [S2, unofficial].

## Background
Holographic display methods for volume data have evolved from early computer-generated holography (CGH) techniques. Early work by Leith and Upatnieks established the communication-theory basis for holography [S1, unofficial]. Modern volumetric holography aims to visualize 3D datasets (medical, scientific) directly in 3D space. The Ray-tracing method for full-parallax CGHs was proposed by Ichikawa et al. to improve realism by handling hidden surfaces and material properties [S1, unofficial]. The Multiple Reference Plane (MRP) approach was introduced by Yan et al. to reduce reconstruction errors in holographic stereograms [S1, unofficial]. Varifocal mirror displays were explored in the context of stereotactic neurosurgery planning to provide 3D visualization of tomographic images [S2, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
