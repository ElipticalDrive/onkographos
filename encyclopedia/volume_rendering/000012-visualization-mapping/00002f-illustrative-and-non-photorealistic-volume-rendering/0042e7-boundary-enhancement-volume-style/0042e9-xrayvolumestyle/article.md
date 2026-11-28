# XRayVolumeStyle
## Overview
XRayVolumeStyle is a non-composable render style used alternatively in the `renderStyle` field of a `VolumeData` node [S1, unofficial]. It serves as a boundary enhancement technique that highlights feature bounds and reduces noise in volume visualization [Parent]. It differs from standard opacity mapping by providing a specific aesthetic for boundary enhancement, distinct from the edge-darkening approach of its child [EdgeEnhancementVolumeStyle](22179-edgeenhancementvolumestyle.md).
## How it works
Inputs and outputs: The method consumes volume data and applies a rendering style configuration. The output is a rendered image where structural boundaries are enhanced.

Data structures: The style is defined within a scene graph node structure. The `renderStyle` field accepts a node of type `XRayVolumeStyle`.

Numbered steps:
1. The `VolumeData` node selects `XRayVolumeStyle` as its active rendering style via the `renderStyle` field [S1, unofficial].
2. The renderer applies the X-Ray style logic to the volume data.
3. The resulting image is composited with other scene elements.

Formulas: Not given in the sources.

Parameters: The style itself does not expose specific parameters in the provided notes. However, the `VolumeData` node allows configuration of `renderSettings` which includes `RayGenerationVolumeSettings` (step size, type) and `AccelerationVolumeSettings` (bounding volume type, traversal function) [S1, unofficial]. These settings can improve image quality and performance but are not specific to XRayVolumeStyle's core definition.

Edge cases and pitfalls: As a non-composable style, it cannot be combined with other render styles in the same `renderStyle` field [S1, unofficial].

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration example based on source structure
volume_data = VolumeData()
volume_data.renderStyle = XRayVolumeStyle()
# Note: Specific internal parameters for XRayVolumeStyle are not detailed in the sources.
# General render settings can be applied to the VolumeData node:
volume_data.renderSettings = [
    RayGenerationVolumeSettings(stepSize=..., type="simple"),
    AccelerationVolumeSettings(boundingVolumeType="boxGeometry", traversalFunction="simple")
]
```
## Subtopics

### [EdgeEnhancementVolumeStyle](22179-edgeenhancementvolumestyle.md)
A non-composable render style that enhances structural edges by darkening voxels based on geometric orientation relative to the view direction. It differs from XRayVolumeStyle by using surface normals to drive visual properties, specifically targeting voxels perpendicular to the camera. Choose it when normal-based edge highlighting is required.

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Strengths and limitations: It is a non-composable style, meaning it must be used exclusively in the `renderStyle` field [S1, unofficial].
When to use it: When a specific boundary enhancement aesthetic is desired within the VolumeData node's rendering pipeline.
## Background
Origin: The method is part of the InstantReality 1.0 framework [S1, unofficial].
History and development: It is documented in the InstantReality tutorial on Volume Rendering [S1, unofficial].
Where it is used: In the InstantReality 1.0 engine for volume visualization [S1, unofficial].
## Sources
[S1] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
