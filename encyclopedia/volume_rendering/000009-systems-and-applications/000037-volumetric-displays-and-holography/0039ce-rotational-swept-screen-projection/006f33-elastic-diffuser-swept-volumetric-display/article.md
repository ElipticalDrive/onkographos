# Elastic Diffuser Swept Volumetric Display

Also called: Elastic Strip Diffuser Array, FlexiVol, FlexiVol Swept Volumetric Display with Elastic Diffuser

## Overview
The Elastic Diffuser Swept Volumetric Display (FlexiVol) is a reach-through volumetric display that replaces the rigid optical diffuser of a planar swept-screen system with an array of elastic strips [S1, S2]. This modification allows users to physically insert their hands into the display volume to interact with true-3D content without distorting the entire display or risking injury [S1, S2]. It sets itself apart from rigid swept displays by enabling direct haptic interaction and from membrane-based displays by isolating deformation to individual strips, thereby preventing global volume distortion and excessive resistive forces [S1].

## How it works
The system builds on [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md) but replaces the rigid screen with a deformable elastic strip array and integrates a hand-tracking pipeline for interaction.

### Inputs and Outputs
*   **Inputs:** User hand position (from cameras), virtual scene geometry, actuation signal phase.
*   **Outputs:** Projected light pattern on the diffuser, rendered 3D hand model within the volume.

### Data Structures and Layouts

**1. Diffuser Strip Array**
*   **Geometry:** Array of parallel strips.
*   **Strip Dimensions:** Width $w = 20$ mm, Length $L = 200$ mm [S1].
*   **Material:** Elastic bands (selected for best compromise of mechanical, optical, and oscillatory properties among tested materials: projection screens, elastane, latex, silicone) [S1, S2].
*   **Mounting:** Cured or attached to a rigid acrylic sheet with a spacer defining the rest thickness [S1].
*   **Elasticity:** Maximum elongation $\Delta L_{max} = 50$ mm (from rest length 185 mm to 235 mm) [S1]. This limit approximates finger push depth on a $190 \times 190 \times 80$ mm volume and prevents irreversible plastic deformation [S1].

**2. Hand Tracking Coordinates**
*   **Source:** Two webcams capturing the interaction space [S2].
*   **Processing:** Video streams processed via MediaPipe for keypoint detection [S2].
*   **Transformation:** 2D image coordinates triangulated and transformed into the volumetric display's 3D coordinate system [S2].
*   **Transmission:** Coordinates sent via UDP to the display rendering software [S2].

**3. Projected Image Buffer**
*   **Format:** 24-bit RGB image [S2].
*   **Content Packing:** Multiple temporal slices are combined into a single spatial frame.
    *   Example: 24 slices with binary color encoding are packed into one 24-bit RGB frame [S2].
*   **Interface:** HDMI or Display Port [S2].

### Numbered Steps

1.  **Fabric Preparation:** Laser-cut elastic materials into $200 \times 20$ mm strips along the elastic dimension [S1]. For curable materials (e.g., silicone), cure sheets of 0.8 mm thickness on an acrylic base with a spacer [S1].
2.  **Hand Tracking & Triangulation:** Capture video from two webcams. Process streams using MediaPipe to extract hand keypoints. Triangulate 3D hand position from the two camera views [S2].
3.  **Coordinate Transformation:** Transform the triangulated hand coordinates from the camera frame to the volumetric display's world coordinate system [S2].
4.  **Synchronization Sampling:** Sample the actuation signal phase (e.g., from a 15 Hz sinusoidal driver) at the start of each projector frame to extrapolate the diffuser's instantaneous position [S1].
5.  **Slice Rendering:** Render the virtual scene slices corresponding to the diffuser's current position. If using binary color projection, reduce color information to binary patterns to multiply effective frame rate [S2].
6.  **Image Packing:** Pack the rendered slices into a single 24-bit RGB image. For example, pack 24 binary slices into the R, G, and B channels [S2].
7.  **Projection:** Send the packed image to the high-frame-rate projector (e.g., Voxon VX1) [S2].
8.  **Hand Visualization:** Render the user's hand as a 3D object within the volume, spatially aligned with the virtual content using the transformed coordinates [S2].
9.  **Interaction Processing:**
    *   **Navigation:** Detect alternate swiping gestures on oscillating bands. If 4 index/middle finger switches are detected, move the camera/environment based on hand orientation. Wrap environment at limits for infinite exploration [S2].
    *   **Touch/Selection:** Detect contact between hand and virtual objects. Trigger events (e.g., bells ringing, balloons popping) [S2].
    *   **Rotation:** Detect grasp (thumb + index/middle). Rotate object based on hand orientation change [S2].
    *   **Deformation:** Detect push on deformable objects. Apply deformation up to a threshold, then split object (e.g., mushrooms popping) [S2].
    *   **Avoidance/Capture:** For dynamic objects (e.g., fish), calculate distance to hand. Move object away faster as distance decreases. If touched and hand stops, trigger interaction animation [S2].

### Formulas

**Projector Frame Rate Requirement**
The required projector operating frequency $f_{proj}$ is determined by the number of slices $N_{slices}$ and the sweep frame rate $f_{sweep}$:
$$ f_{proj} = N_{slices} \times f_{sweep} $$
*   Example: 100 slices at 30 Hz sweep requires 3000 Hz projector operation [S2].

**Effective Frame Rate with Binary Color**
When using binary pattern multiplication, the effective frame rate $f_{eff}$ is:
$$ f_{eff} = f_{proj} \times N_{bitplanes} $$
*   Example: Voxon VX1 at 120 Hz with 24-bit color projects binary patterns at 2880 Hz (implying $N_{bitplanes}$ logic or specific time-multiplexing scheme described in source) [S2]. Note: The source states "projects binary patterns at 2880 Hz" for a 120 Hz base with 24-bit color, which implies a specific multiplexing factor or that the 2880 Hz is the *result* of the binary projection strategy on the hardware limit. The source explicitly says: "For example, the Voxon VX1 projects 192 slices at 15 Hz (2880 Hz)... When operating at 120 Hz with 24-bit color depth, it projects binary patterns at 2880 Hz" [S2]. This suggests the hardware limit or the binary projection constraint caps the effective slice rate.

**Strip Deformation Approximation**
The deformation of a single strip can be approximated by a sinusoid or parabola, unlike 2D membrane deformation which is complex and global [S1].

### Parameters and Typical Values

*   **Strip Width:** 20 mm [S1]. Chosen to match finger width. Smaller strips cause optical discontinuities; larger strips behave like membranes [S1].
*   **Rest Length:** 185 mm [S1].
*   **Max Elongation:** 50 mm (Total length 235 mm) [S1].
*   **Display Volume:** $190 \times 190 \times 80$ mm [S1].
*   **Projector:** Voxon VX1 (DLP) [S2].
*   **Actuation Signal:** 15 Hz sinusoidal [S1].
*   **Projector VSync:** 120 Hz [S1].
*   **Slices:** 192 slices at 15 Hz sweep (2880 Hz effective) [S2].
*   **Resolution:** 980x720 px (at 15 Hz) or 1280x800 px (at 120 Hz binary) [S2].

### Edge Cases and Pitfalls

*   **Optical Discontinuities:** Using strips narrower than 20 mm creates visible gaps or artifacts between strips [S1].
*   **Plastic Deformation:** Exceeding the 50 mm elongation limit causes irreversible damage to the elastic material [S1].
*   **Synchronization Drift:** Even with matched frequencies, drift occurs. The rendering system must sample the actuation signal phase every frame to extrapolate position [S1].
*   **Force Feedback:** Membranes exert high anisotropic forces on fingers; strips mitigate this but still require careful material selection to avoid discomfort [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# FlexiVol Interaction and Rendering Loop

def main_loop():
    # Initialization
    diffuser = ElasticStripArray(width=20.0, length=200.0, max_elongation=50.0)
    projector = DLPProjector(model="Voxon_VX1", resolution=(1280, 800))
    hand_tracker = MediaPipeHandTracker(camera_count=2)
    scene = VirtualScene()
    
    # Synchronization setup
    actuation_signal = get_actuation_signal() # 15 Hz sinusoid
    vsync_freq = 120.0 # Hz
    
    while True:
        # 1. Hand Tracking
        hand_keypoints = hand_tracker.get_keypoints()
        if hand_keypoints is not None:
            # Triangulate and transform to display coordinates
            hand_pos_3d = triangulate_and_transform(hand_keypoints)
        else:
            hand_pos_3d = None
            
        # 2. Synchronization
        # Sample actuation signal phase to determine diffuser position
        phase = actuation_signal.get_phase()
        diffuser_position = extrapolate_diffuser_position(phase)
        
        # 3. Interaction Logic
        interaction_events = []
        if hand_pos_3d is not None:
            # Navigation: Alternate swiping detection
            if detect_alternate_swipe(hand_keypoints):
                nav_delta = calculate_orientation_change(hand_keypoints)
                scene.navigate(nav_delta)
                
            # Touch/Selection
            for obj in scene.objects:
                if is_touching(hand_pos_3d, obj):
                    trigger_event(obj, "touch")
                    
            # Rotation
            if is_grasping(hand_keypoints):
                rotation_delta = calculate_grasp_rotation(hand_keypoints)
                scene.rotate_active_object(rotation_delta)
                
            # Deformation
            for obj in scene.deformable_objects:
                if is_pushing(hand_pos_3d, obj):
                    deformation = apply_push(hand_pos_3d, obj)
                    if deformation > threshold:
                        obj.split()
                        
            # Dynamic Object Interaction (Fish)
            for fish in scene.fish:
                dist = distance(hand_pos_3d, fish.pos)
                if dist < avoidance_radius:
                    fish.avoid(hand_pos_3d, dist)
                elif is_touching(hand_pos_3d, fish) and hand_is_still(hand_keypoints):
                    fish.interact()

        # 4. Rendering
        # Render slices based on diffuser position
        slices = scene.render_slices(diffuser_position)
        
        # Pack slices into binary color image
        # Example: 24 slices packed into 24-bit RGB
        packed_image = pack_slices_into_rgb(slices, num_slices=24)
        
        # 5. Projection
        projector.display(packed_image)
        
        # 6. Hand Visualization
        if hand_pos_3d is not None:
            render_hand_model(hand_pos_3d)
```

## Subtopics

### Holographic Defuser Light Field Display
Uses holographic principles for light field generation, differing from the swept-scan mechanical approach. [Link](7245-holographic-defuser-light-field-display.md)

### Cathode Ray Volumetric Display
Uses electron beams on phosphor screens, lacking the direct reach-through interaction of elastic diffusers. [Link](7311-cathode-ray-volumetric-display.md)

### Seelinder
A specific type of volumetric display, distinct in its optical path and interaction model. [Link](7312-seelinder.md)

### Dynamic Projection Mapping for Rotating Volumetric Display
Focuses on projection mapping techniques for rotating screens, potentially using rigid screens. [Link](7619-dynamic-projection-mapping-for-rotating-volumetric-display.md)

### Oscillating Diffuser Volumetric Display
Uses oscillation instead of rotation, but may use rigid diffusers. [Link](13163-oscillating-diffuser-volumetric-display.md)

### Radial Multiview Three-Dimensional Display
Uses multiple views rather than temporal slicing on a single screen. [Link](20670-radial-multiview-three-dimensional-display.md)

### Viewpoint-surrounding Hemisphere System
Uses a hemisphere screen, differing in form factor and interaction space. [Link](21572-viewpoint-surrounding-hemisphere-system.md)

### Rotating Rigid Screen with Projector
The base parent method, using a non-deformable screen. [Link](22705-rotating-rigid-screen-with-rotating-volumetric-display.md)

## Operating characteristics

*   **Time Complexity:** Rendering is linear with the number of slices $N_{slices}$. Hand tracking is constant time per frame but requires significant CPU/GPU for MediaPipe processing.
*   **Memory Complexity:** Proportional to the number of slices and image resolution.
*   **Performance:** The Voxon VX1 projector can handle up to 2880 Hz effective frame rate with binary color projection [S2].
*   **Strengths:** Enables direct reach-through interaction; isolates deformation to local areas; avoids global volume distortion; safe for users.
*   **Limitations:** Optical discontinuities between strips; limited deformation range (50 mm); requires precise synchronization; complex interaction logic for dynamic objects.
*   **When to Use:** Applications requiring direct physical interaction with 3D content, such as education, training, or playful exploration.
*   **When Not to Use:** Applications requiring high optical uniformity without strip artifacts, or where large-scale deformation of the display volume is needed.

## Background

The Elastic Diffuser Swept Volumetric Display, known as FlexiVol, was introduced to address the lack of direct interaction in traditional swept volumetric displays, which use rigid diffusers [S1, S2]. The concept was developed by researchers presenting at CHI EA '26 in Barcelona, Spain [S2]. The work builds on previous swept volumetric display technologies, replacing the rigid screen with an array of elastic strips to enable "come-and-interact" scenarios [S2]. The design choice of strips over membranes was driven by the need to prevent global volume distortion and excessive resistive forces on the user's fingers [S1].

## Sources

[S1] FlexiVol: a Volumetric Display with an Elastic Diffuser to Enable Reach-Through Interaction, https://doi.org/10.1145/3706598.3714315, fetched 2026-10-26
[S2] Direct Interaction with a Reach-through Volumetric Display that uses an Elastic Diffuser, https://doi.org/10.1145/3772363.3799374, fetched 2026-10-29
