# Rotational Swept-Screen Projection

## Overview
Rotational swept-screen projection is a class of volumetric display systems that generates a continuous 3D image by projecting 2D image patterns onto a screen undergoing rotational motion [S1, unofficial]. It solves the problem of creating a stationary, viewable 3D volume without requiring the light source itself to scan the entire volume mechanically, instead relying on the temporal integration of the human eye as the screen sweeps through space [S1, unofficial]. It differs from translational swept-screen methods by utilizing rotational kinematics to define the volume, and from static multi-layer displays by physically moving the projection surface [S1, unofficial].

## Shared principles
The core mechanism involves a screen (reflective, transmissive, or emissive) rotating around a central axis [S1, unofficial]. A projection system (LED array, laser, or projector) updates the image on the screen synchronously with the rotation angle [S1, unofficial].

1. **Coordinate Mapping**: The 3D world coordinates of a voxel are mapped to 2D screen coordinates and a specific frame index corresponding to the rotation angle [S1, unofficial].
2. **Temporal Integration**: The screen speed and frame rate are chosen such that the persistence of vision blends the discrete 2D slices into a continuous 3D volume [G].
3. **Synchronization**: Precise timing between the screen's angular position and the image update is critical [G]. Misalignment between screen position and projected frame index results in Z-depth distortion [8515-multi-planar-volumetric-3d-display.md].

## Subtopics

### [Volumetric Display using Rotating Mirror (VRDM)](2567-volumetric-display-using-rotating-mirror-vrdm.md)
VRDM projects 3D voxel data onto a rotating mirror using surrounding LED screens, mapping 3D coordinates to 2D screen pixels via analytical mirror reflection matrices. It differs from standard ray marching by avoiding per-pixel ray tracing loops, instead computing direct projections for each rotation angle to drive the LED arrays synchronously. Choose this method when a reflective, pyramid-shaped volume is desired with high geometric precision.

### [2D Full-Color LED Array Rotational Display](7246-2d-full-color-led-array-rotational-display.md)
This system renders volumetric images by rotating a 2D LED panel array around a central axis and synchronously updating the panel content to match the rotation angle. It differs from laser-based swept-screen systems by generating voxels directly on an emissive surface without requiring external synchronization with electron beams or lasers. Choose this method for direct emissive volumetric rendering with high frame counts per rotation.

### [Multi-planar volumetric 3D display](8515-multi-planar-volumetric-3d-display.md)
This method projects a sequence of 2D image patterns onto a screen moving along the Z-axis to form a stack of spatial image planes, creating a true 3D image viewable without glasses. Unlike conventional 2D screens that rely on psychological depth cues, this method places voxels at their physical spatial positions, emitting light omni-directionally to provide both physiological and psychological depth cues. It distinguishes itself from sibling methods like the Volumetric Display using Rotating Mirror (VRDM) by utilizing linear or reciprocating screen motion rather than rotational sweep. Choose this method for planar volumetric displays requiring precise Z-depth layering.

### Cathode Ray Volumetric Display
A volumetric display system that generates voxels by irradiating a fluorescent screen coated on a rotating plane with an electron beam [S1, unofficial]. This is an early implementation of the rotational swept-screen principle, predating laser and LED variants [S1, unofficial].

### Seelinder
A cylindrical 3D display using a parallax barrier with multiple slits and a 1D LED array rotating along the slits [S1, unofficial]. It utilizes the scanning of light rays when slit and LED positions change to reproduce multiple viewpoints in 360 degrees [S1, unofficial].

### Oscillating Diffuser Volumetric Display
A 360-degree display using a moving projection surface (vertically oscillating diffuser) synchronized with a projection system [S2]. It relies on retinal persistence to create a full volumetric object [S2]. The system oscillates a diffuser vertically at high frequency, synchronizes the projection system with the diffuser's motion, and the viewer perceives the image as a full object due to retinal persistence [S2]. The high frequency precludes the diffuser from being seen, leading to general transparency [S2]. Occlusion from different parts of the image is not possible because the image is diffused in a large field of view, limiting the display to 'ghosted content' rather than true volumetric occlusion [S2].

## Background
The concept of rotational volumetric displays dates back to the 1950s and 1960s [S1, unofficial]. In 1958, Withey proposed a system using a translating fluorescent screen inside a vacuum container [S1, unofficial]. In the 1960s, Hartwig proposed laser irradiation of rotating screens, followed by a helical screen structure [S1, unofficial]. In 1994, Blundell et al. proposed a system based on generating an image drawing space, and the "Cathode ray sphere" was proposed, using a 15 Hz rotating fluorescent plane [S1, unofficial]. The 2000s saw a shift towards LED and laser excitation, with Hisatake proposing the rotation of lanthanide-mixed polymer sheets excited by a 410 nm laser [S1, unofficial]. In 2004, Perlin et al. demonstrated volumetric imaging by irradiating dust in the air with lasers [S1, unofficial]. In 2010, the Seelinder system was proposed, using a cylindrical parallax barrier with a rotating 1D LED array [S1, unofficial]. Recent developments include holographic diffuser combinations that form multiple viewpoints in 360 degrees [S1, unofficial].

## Sources
[S1] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
[S2] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
