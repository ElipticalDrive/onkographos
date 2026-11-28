# Rotating Rigid Screen with Projector

Also called: Rotational Swept-Screen Projection with Projector

## Overview
This method renders volumetric images by rotating a rigid reflective or transmissive screen around a central axis while a projector synchronously updates the 2D image content to match the rotation angle [S1, unofficial]. It distinguishes itself from sibling methods like the [Holographic Defuser Light Field Display](7245-holographic-defuser-light-field-display.md) and [Cathode Ray Volumetric Display](7311-cathode-ray-volumetric-display.md) by using a continuous rotational sweep rather than linear reciprocating motion or electron beam deflection, allowing for higher frame rates and larger display scales [S1, unofficial].

## Shared principles
The core mechanism involves a screen (reflective, transmissive, or emissive) rotating around a central axis [S1, unofficial]. A projection system (LED array, laser, or projector) updates the image on the screen synchronously with the rotation angle [S1, unofficial].

1.  **Coordinate Mapping**: The 3D world coordinates of a voxel are mapped to 2D screen coordinates and a specific frame index corresponding to the rotation angle [S1, unofficial].
2.  **Temporal Integration**: The screen speed and frame rate are chosen such that the persistence of vision blends the discrete 2D slices into a continuous 3D volume [G].
3.  **Synchronization**: Precise timing between the screen's angular position and the image update is critical [G]. Misalignment between screen position and projected frame index results in Z-depth distortion [8515-multi-planar-volumetric-3d-display.md].

## Subtopics

### [2D Full-Color LED Array Rotational Display](7246-2d-full-color-led-array-rotational-display.md)
A 2D full-color LED array rotational display renders volumetric images by rotating a 2D LED panel array around a central axis and synchronously updating the panel content to match the rotation angle. It differs from laser-based swept-screen systems by generating voxels directly on an emissive surface without requiring external synchronization with electron beams or lasers. See [2D Full-Color LED Array Rotational Display](7246-2d-full-color-led-array-rotational-display.md).

### [Multi-planar volumetric 3D display](8515-multi-planar-volumetric-3d-display.md)
A multi-planar volumetric 3D display projects a sequence of 2D image patterns onto a screen moving along the Z-axis to form a stack of spatial image planes, creating a true 3D image viewable without glasses. Unlike conventional 2D screens that rely on psychological depth cues, this method places voxels at their physical spatial positions, emitting light omni-directionally to provide both physiological and psychological depth cues. Builds on [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md). It distinguishes itself from sibling methods like [Volumetric Display using Rotating Mirror (VRDM)](2567-volumetric-display-using-rotating-mirror-vrdm.md), [2D Full-Color LED Array Rotational Display](7246-2d-full-color-led-array-rotational-display.md), and [Oscillating Swept Volume Display](15668-oscillating-swept-volume-display.md) by utilizing linear or reciprocating screen motion rather than rotational sweep [G]. See [Multi-planar volumetric 3D display](8515-multi-planar-volumetric-3d-display.md).

### Rotating Volumetric Display
**Definition**: A larger volumetric display variant using a rotating screen resembling a drill bit with ramps and drop-offs, and projectors positioned above, allowing for higher frame rates and larger projections [S1, unofficial].

**Steps**:
1.  Use a helical slicing algorithm instead of a linear slicing algorithm to process the 3D form [S1, unofficial].
2.  Align three projectors (red, green, blue) into the same space [S1, unofficial].
3.  Perform keystone calibration to align the projectors [S1, unofficial].

**Pitfalls**:
1.  A dead spot appears in the middle of the image where the surface is not moving fast enough to maintain the illusion, appearing as a vortex-like hole [S1, unofficial].
2.  Keystone calibration complexity can result in pink fringing around icons [S1, unofficial].

**Comparison**: The rotating display delivers higher frame rates compared to the reciprocating display, but is harder to scale up due to the difficulty of moving the glass fast enough for a solid illusion in larger sizes [S1, unofficial].

## Background
Originating from work by Voxon Photonics and the Silverton team, this approach was developed to address the scaling limitations of reciprocating "glass shaker" volumetric displays [S1, unofficial]. The rotating design allows for larger physical dimensions and higher frame rates, though it introduces complexities in multi-projector alignment and helical slicing [S1, unofficial].

## Sources
[S1] Voxon hologram volumetric display (2019), https://newatlas.com/vr/voxon-photonics-3d-hologram-volumetric-displays/, fetched 2026-10-26 (unofficial)
