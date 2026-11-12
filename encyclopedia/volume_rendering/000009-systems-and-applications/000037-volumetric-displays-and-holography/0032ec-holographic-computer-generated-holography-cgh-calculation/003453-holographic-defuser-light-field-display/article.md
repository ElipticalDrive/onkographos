# Holographic Defuser Light Field Display

Also called: Jones et al. holographic rotating diffuser, Jones light-field display

## Overview
The Holographic Defuser Light Field Display is a volumetric display system that generates a 360° horizontal field of view with multiple distinct viewpoints by synchronizing a high-speed projector with a rotating holographic defuser [S1][S2, unofficial]. It solves the occlusion limitation of simple diffusing displays by restricting the reflection angle of the defuser, allowing viewers at different horizontal positions to see different images [S1]. Unlike sibling methods that rely on mechanical scanning screens or static parallax barriers, this method uses the angular selectivity of a holographic element to spatially multiplex viewpoints in free space [S2, unofficial].

## How it works
**Inputs:**
- Video frame buffer containing $N$ distinct viewpoint images.
- Rotation speed $\omega$ of the holographic defuser.
- Synchronization signal for projector update timing.

**Outputs:**
- A 360° light field where each horizontal angle $\theta$ corresponds to a specific viewpoint image.

**Data Structures:**
- `ViewpointBuffer`: Array of $N$ images, where $N$ is the number of viewpoints.
- `DefuserAngle`: Current rotation angle of the diffuser, $\alpha(t)$.

**Parameters:**
- $N = 288$: Number of images generated [S1].
- $\Delta \theta_{ref} = 1.25^\circ$: Reflection angle range of the holographic defuser [S2, unofficial].
- $\Theta_{FOV} = 360^\circ$: Horizontal field of view [S1].

**Steps:**
1. Initialize the holographic defuser to rotate at constant angular velocity $\omega$.
2. Calculate the current azimuthal angle of the defuser's normal vector relative to the viewer's position.
3. Determine which viewpoint index $i$ corresponds to the current defuser orientation. Since the defuser reflects light within a narrow cone of $\Delta \theta_{ref}$, only the image currently aligned with the viewer's line of sight is visible.
4. Update the projector with the image from `ViewpointBuffer[i]`.
5. Synchronize the image update rate with the rotation speed such that the image changes exactly when the defuser rotates to the next viewpoint sector.
6. Repeat for continuous rotation.

**Edge Cases and Pitfalls:**
- **Occlusion:** The system does not support occlusion between viewers or objects because the image is diffused over a large field of view for each viewpoint [S1]. It is limited to "ghosted content" where objects do not hide each other [S1].
- **Viewer Position:** Viewers must be at a specific height and distance from the device to align with the reflection cone [S1].
- **Precision:** The synchronization between projector refresh and defuser rotation must be precise to avoid image tearing or misalignment of viewpoints [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_VIEWPOINTS = 288
REFLECTION_ANGLE_DEG = 1.25
FOV_DEG = 360
VIEWPOINT_SPACING_DEG = FOV_DEG / NUM_VIEWPOINTS

# State
defuser_angle = 0.0
current_viewpoint_index = 0

def update_projector(viewpoint_index):
    """Send the image for the given viewpoint index to the projector."""
    image = ViewpointBuffer[viewpoint_index]
    projector.display(image)

def main_loop(rotation_speed_deg_per_sec, frame_rate):
    """
    Synchronize defuser rotation with projector updates.
    """
    while True:
        # Update defuser angle based on time
        dt = 1.0 / frame_rate
        defuser_angle = (defuser_angle + rotation_speed_deg_per_sec * dt) % FOV_DEG
        
        # Calculate which viewpoint is currently aligned with the viewer
        # The defuser reflects light within a narrow cone.
        # We assume the viewer is at a fixed position and the defuser rotates.
        # The visible viewpoint is determined by the defuser's orientation.
        
        # Map defuser angle to viewpoint index
        # Each viewpoint covers a sector of VIEWPOINT_SPACING_DEG
        # The reflection cone is 1.25 degrees, which is smaller than the spacing
        # (360/288 = 1.25 degrees). This implies a tight fit.
        
        new_viewpoint_index = int(defuser_angle / VIEWPOINT_SPACING_DEG) % NUM_VIEWPOINTS
        
        if new_viewpoint_index != current_viewpoint_index:
            current_viewpoint_index = new_viewpoint_index
            update_projector(current_viewpoint_index)
```

## Subtopics

### Holographic Computer-Generated Holography (CGH) Calculation
The parent method providing the theoretical basis for wavefront calculation. [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md)

### Volume Rendering based CGH Calculation
Uses voxel-based wavefront superposition. [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md)

### Diffraction-Specific Approach
Focuses on diffraction effects in holographic elements. [Diffraction-Specific Approach](7015-diffraction-specific-approach.md)

### Holographic Stereogram
Generates 3D images from a series of 2D images. [Holographic Stereogram](7067-holographic-stereogram.md)

### Computational Phase Modulation (CGH Generation)
Uses phase-only modulation for CGH. [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md)

### Inverse Design Algorithm for Voxel-based CGVHs
Optimizes voxel positions for holographic displays. [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md)

### Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)
Uses multiple planes for HPO displays. [Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)](10810-multiple-wavefront-recording-planes-wrp-with-horizontal-para.md)

## Operating characteristics
- **Time Complexity:** The computational load is dominated by the projector's frame rate and the synchronization logic, which is $O(1)$ per frame. The optical generation is real-time.
- **Memory Complexity:** Requires storage for $N$ viewpoint images. For $N=288$, this is significant but manageable.
- **Performance:** Generates 288 viewpoints on a 360° horizontal field of view [S1].
- **Strengths:** Allows multi-user sharing of the visual experience at a given height and distance [S1]. No moving parts in the optical path other than the defuser.
- **Limitations:** No occlusion between objects or viewers [S1]. Limited to ghosted content. Viewer position is constrained [S1].
- **When to use:** When a 360° multi-user volumetric display is needed without occlusion.
- **When not to use:** When occlusion is required or when viewers need to move freely in 3D space.

## Background
The method was proposed by Jones et al. to allow multi-user sharing of the visual experience at a given height and distance from the device [S1]. It uses a holographic rotating diffuser with specific diffusing angular characteristics to generate a light-field display [S1]. The system generates 288 images on a 360° horizontal field of view [S1]. The holographic defuser reflects images within a limited range of 1.25 degrees, updating the image synchronized with rotation to generate viewpoints [S2, unofficial]. This approach is a type of scanning volumetric display, similar to earlier systems by Parker and Wallis (1948) and Withey (1958), but uses holographic elements for angular selectivity [S2, unofficial].

## Sources
[S1] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
