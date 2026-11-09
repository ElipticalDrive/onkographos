# iVR-GS Relighting via Gaussian Primitives

Also called: iVR-GS

## Overview
iVR-GS is a novel view synthesis (NVS) framework for volume visualization (VolVis) scenes that leverages composable and editable 3D Gaussian primitives to enable flexible exploration without direct access to volume data [S1]. It distinguishes itself from baseline 3D Gaussian Splatting (3DGS) and NeRF-based methods (Plenoxels, CCNeRF) by explicitly modeling lighting effects via the Blinn-Phong model, allowing for high-quality relighting and scene editing [S1]. Unlike simple image-space interpolation, iVR-GS reconstructs underlying geometry to ensure physically plausible relighting results [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input views of a VolVis scene. [S1]
*   **Outputs:** Relit images of the scene under varying light source polar angles. [S1]

**Data Structures:**
The method represents the scene using 3D Gaussian primitives. [S1] The exact parameterization of each primitive is not given in the sources. [G]

**Method Steps:**
1.  **Light Source Configuration:** Fix the azimuthal angle of the light source to be identical to the camera's azimuthal angle. [S1]
2.  **Polar Angle Adjustment:** Vary the polar angle of the light source from $-90^\circ$ (bottom) to $90^\circ$ (top). [S1]
3.  **Sampling:** Uniformly sample 180 images along this light source trajectory. [S1]
4.  **Rendering:** Render the scene using the Blinn-Phong lighting model applied to the Gaussian primitives. [S1]

**Formulas:**
The lighting effect is modeled using the Blinn-Phong model. [S1] The specific mathematical formulation for the Blinn-Phong shading is not detailed in the provided text, but it is applied to the Gaussian primitives to compute the final color. [S1]

**Parameters:**
*   Number of sampled images: 180. [S1]
*   Polar angle range: $[-90^\circ, 90^\circ]$. [S1]

**Edge Cases and Pitfalls:**
*   **Lighting Model Mismatch:** iVR-GS relies on the Blinn-Phong model. It may encounter challenges when handling VolVis scenes that employ different lighting models, potentially requiring a redesign of the lighting representation. [S1]
*   **Interpolation Failure:** Linear interpolation (LERP) of the first and last relit images fails to produce faithful results because it ignores geometry. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def linspace(start, stop, num):
    """Return evenly spaced numbers over a specified interval."""
    if num == 1:
        return [start]
    step = (stop - start) / (num - 1)
    return [start + i * step for i in range(num)]

def spherical_to_cartesian(azimuth_deg, polar_deg):
    """Convert spherical angles to a unit direction vector.
    Azimuth is in the XY plane, polar is elevation from XY plane."""
    az_rad = math.radians(azimuth_deg)
    pol_rad = math.radians(polar_deg)
    cos_pol = math.cos(pol_rad)
    return (cos_pol * math.cos(az_rad), cos_pol * math.sin(az_rad), math.sin(pol_rad))

def relight_ivrgs(gaussians, camera_azimuth, num_samples=180):
    """Relight a scene represented by 3D Gaussian primitives using iVR-GS."""
    images = []
    polar_angles = linspace(-90.0, 90.0, num_samples)
    for polar_angle in polar_angles:
        light_dir = spherical_to_cartesian(camera_azimuth, polar_angle)
        # The specific Blinn-Phong shading formula and Gaussian rasterization pipeline are not given in the sources. [G]
        image = None
        images.append(image)
    return images
```

## Subtopics

### [Editable Gaussian Representation](7495-editable-gaussian-representation.md)
Extends the framework by parameterizing each Gaussian with explicit shading attributes and using a palette-based color representation for decoupled geometry and appearance editing.

### [Relightable Gaussian Codec Avatars](1482-relightable-gaussian-codec-avatars.md)
The parent method introducing relightable Gaussian avatars supporting diverse lighting conditions.

## Operating characteristics
*   **Quality:** Produces higher-quality reconstruction results compared to Plenoxels, CCNeRF, and base 3DGS. [S1]
*   **Real-time:** Supports real-time scene editing and exploration. [S1]
*   **Limitations:** The reliance on the Blinn-Phong model limits its applicability to scenes with different lighting models. [S1]
*   **Comparison:** Outperforms linear interpolation (LERP) for relighting tasks by modeling geometry, whereas LERP fails to produce faithful results. [S1]

## Background
iVR-GS (Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting) was presented as a novel NVS framework for VolVis scenes. [S1] It addresses limitations in existing NVS methods by leveraging composable and editable Gaussian primitives. [S1] The method was evaluated on datasets such as "chameleon" and "supernova". [S1]

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
