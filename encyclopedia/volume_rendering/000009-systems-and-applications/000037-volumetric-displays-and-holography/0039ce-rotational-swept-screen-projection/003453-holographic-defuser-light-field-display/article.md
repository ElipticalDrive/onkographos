# Holographic Defuser Light Field Display

Also called: Jones et al. holographic rotating diffuser, Jones light-field display

## Overview
A volumetric display system that generates a 360° horizontal light field by projecting images onto a rotating holographic diffuser screen [S1][S2, unofficial]. Unlike standard rotational swept-screen displays that diffuse light broadly, this method uses a diffuser with specific angular characteristics to reflect light within a narrow cone (1.25°), creating distinct viewpoints for multiple users simultaneously [S1][S2, unofficial]. It solves the limitation of occlusion in traditional ghosted volumetric displays, though it remains constrained to content without complex occlusion [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of 2D images corresponding to specific viewing angles; rotation speed of the screen; projector frame rate.
*   **Output:** A 360° horizontal light field with 288 distinct viewpoints [S1].

**Data Structures:**
*   **Image Buffer:** A sequence of $N=288$ images [S1]. Each image corresponds to a specific angular sector of the 360° field of view.
*   **Screen Geometry:** A cylindrical or planar surface coated with a holographic diffuser material [S1][S2, unofficial].
*   **Diffuser Property:** A reflection profile limited to a range of $\theta_{max} = 1.25^\circ$ [S2, unofficial].

**Steps:**
1.  **Synchronization Setup:** Synchronize the projector's frame update with the angular position of the rotating screen [S2, unofficial]. The screen rotates continuously around a central vertical axis.
2.  **Image Selection:** For each projector frame $k$, select the image $I_k$ corresponding to the current viewing angle $\phi_k$ determined by the screen's rotation angle [S1][S2, unofficial].
3.  **Projection:** Project image $I_k$ onto the screen surface [S2, unofficial].
4.  **Diffusion:** The holographic diffuser reflects the projected light only within the narrow angular range of $\pm 0.625^\circ$ (centered on the specular reflection direction) [S2, unofficial].
5.  **Viewpoint Generation:** As the screen rotates, the narrow reflection cone sweeps across the audience, delivering the correct perspective image to each viewer's position [S1][S2, unofficial].
6.  **Temporal Integration:** The persistence of vision blends the discrete viewpoints into a continuous 3D light field experience for multiple users [S1][S2, unofficial].

**Formulas:**
*   **Viewpoint Count:** $N = 288$ [S1].
*   **Angular Resolution:** The angular width of each viewpoint is determined by the diffuser's reflection range, $\Delta \theta = 1.25^\circ$ [S2, unofficial].
*   **Mapping:** The image index $k$ is mapped to the rotation angle $\alpha$ such that $k = \text{round}(\frac{\alpha}{360^\circ} \times 288)$ [S1][S2, unofficial].

**Parameters:**
*   **Number of Images:** 288 [S1].
*   **Horizontal Field of View:** 360° [S1][S2, unofficial].
*   **Reflection Angle Range:** 1.25° [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion:** The system cannot render occlusion between objects in the scene because the light is diffused into a wide horizontal field of view for all viewers simultaneously [S1]. It is limited to "ghosted content" where objects do not block each other from different viewpoints [S1].
*   **Viewer Position:** Users must be at a specific height and distance from the device to receive the correct viewpoint [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Holographic Defuser Light Field Display Rendering
# N is the number of viewpoints (288)
# theta_max is the diffuser reflection half-angle (0.625 degrees)
# total_fov is the horizontal field of view (360 degrees)

def render_holographic_display(images, rotation_angle_degrees):
    """
    Selects and projects the correct image based on screen rotation.
    
    Args:
        images: List of 288 pre-rendered 2D images.
        rotation_angle_degrees: Current angle of the screen (0-360).
        
    Returns:
        The image to project.
    """
    # Map rotation angle to image index
    # Ensure angle is normalized to 0-360
    normalized_angle = rotation_angle_degrees % 360.0
    
    # Calculate index
    # 288 images cover 360 degrees
    index = int((normalized_angle / 360.0) * 288)
    
    # Handle boundary case if index equals 288
    if index >= 288:
        index = 0
        
    return images[index]

def check_diffuser_constraint(viewer_angle_degrees, screen_normal_angle_degrees):
    """
    Checks if a viewer is within the narrow reflection cone.
    
    Args:
        viewer_angle_degrees: Angle of the viewer relative to the screen center.
        screen_normal_angle_degrees: Angle of the screen's surface normal.
        
    Returns:
        True if viewer is in the 1.25 degree reflection cone.
    """
    # Calculate difference between viewer and specular reflection direction
    angle_diff = abs(viewer_angle_degrees - screen_normal_angle_degrees)
    
    # The reflection range is 1.25 degrees total
    # So the viewer must be within 0.625 degrees of the specular direction
    if angle_diff <= 0.625:
        return True
    else:
        return False
```

## Subtopics

### Volumetric Display using Rotating Mirror (VRDM)
Uses a physical mirror to sweep a laser or light source. Offers true occlusion but is limited to line-based or sparse volumetric rendering. [Link](2567-volumetric-display-using-rotating-mirror-vrdm.md)

### 2D Full-Color LED Array Rotational Display
Uses an array of LEDs on a rotating ring. Simpler hardware but limited resolution and no true light-field perspective correction. [Link](7246-2d-full-color-led-array-rotational-display.md)

### Dynamic Projection Mapping for Rotating Volumetric Display
Projects onto a generic rotating screen. Lacks the angular selectivity of the holographic defuser, resulting in ghosting for multiple viewers. [Link](7619-dynamic-projection-mapping-for-rotating-volumetric-display.md)

### Multi-planar volumetric 3D display
Uses discrete planes. Does not provide continuous 3D depth or 360° light field. [Link](8515-multi-planar-volumetric-3d-display.md)

### Oscillating Swept Volume Display
Oscillates rather than rotates. Similar projection challenges but limited to a 180° or less field of view. [Link](15668-oscillating-swept-volume-display.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per frame for image selection, assuming pre-computed viewpoints. The bottleneck is the projector's frame rate and the mechanical rotation speed.
*   **Memory Complexity:** $O(N)$ where $N=288$ is the number of viewpoints. Each viewpoint must be stored as a 2D image.
*   **Measured Performance:** Generates 288 images on a 360° horizontal field of view [S1]. The reflection angle is limited to 1.25° [S2, unofficial].
*   **Strengths:** Allows multi-user sharing of the visual experience at a given height and distance [S1]. Provides a true 360° light field with distinct viewpoints.
*   **Limitations:** Cannot render occlusion between objects, limiting content to "ghosted" scenes [S1]. Requires precise synchronization between rotation and projection [S2, unofficial].
*   **When to use:** For multi-user 360° displays where occlusion is not critical.
*   **When not to use:** For scenes requiring complex depth interactions or occlusion.

## Background
The system was proposed by Jones et al. to allow multi-user sharing of the visual experience at a given height and distance from the device [S1]. It addresses the limitation of traditional rotational swept-screen displays where broad diffusion prevents distinct viewpoints for multiple users [S1]. The concept builds on earlier work by Joens et al. in 2007, who combined a rotating holographic defuser with a high-speed projector to form multiple viewpoints in 360 degrees [S2, unofficial].

## Sources
[S1] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
