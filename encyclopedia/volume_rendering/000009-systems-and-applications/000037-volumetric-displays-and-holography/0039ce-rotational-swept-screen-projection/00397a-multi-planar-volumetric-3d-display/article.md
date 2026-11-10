# Multi-planar volumetric 3D display

Also called: Moving screen volumetric display, Swept-screen volumometric display, Temporal integration volumetric display

## Overview
A multi-planar volumetric 3D display is a hardware-based rendering technique that projects a sequence of 2D image patterns onto a screen moving along the Z-axis to form a stack of spatial image planes, creating a true 3D image viewable without glasses [S1]. Unlike conventional 3D rendering on flat screens which relies on psychological depth cues and brain interpretation, this method physically places voxels at their spatial positions, emitting light omni-directionally to provide both physiological and psychological depth cues [S1]. It is distinguished from its sibling [Volumetric Display using Rotating Mirror (VRDM)](2567-volumetric-display-using-rotating-mirror-vrdm.md) by using a planar moving screen rather than a rotating reflective surface to define the volume.

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of $N$ 2D image frames corresponding to different Z-depths; a control signal for the screen actuator.
*   **Output:** A continuous 3D volumetric image perceived by the human eye.

**Data Structures:**
*   **Frame Buffer:** Stores $N$ 2D image patterns. Each pattern corresponds to a specific Z-plane position within the display volume.
*   **Screen Position State:** Tracks the current Z-coordinate of the moving screen.

**Parameters:**
*   $f_{sweep}$: Screen sweep frequency. Must be $> 24$ Hz to ensure proper temporal integration and avoid flicker [S1].
*   $N$: The number of 2D image frames projected during one complete sweep cycle [S1].
*   $T_{cycle}$: Duration of one complete sweep cycle ($T_{cycle} = 1/f_{sweep}$).

**Steps:**
1.  **Initialize Screen:** Position the screen at the start of its Z-axis travel range.
2.  **Start Sweep:** Move the screen back and forth along the Z direction at frequency $f_{sweep}$ [S1].
3.  **Synchronize Projection:** For each complete sweep cycle (duration $T_{cycle}$):
    a.  Iterate through the $N$ pre-computed 2D image frames $i = 0 \dots N-1$.
    b.  Calculate the required Z-position $Z_i$ for frame $i$ based on the desired volume depth and the screen's current trajectory.
    c.  Project the $i$-th 2D image pattern onto the screen at position $Z_i$ [S1].
    d.  Ensure the projection timing aligns with the screen's velocity so that the image is captured at the correct spatial slice.
4.  **Repeat:** Continue the sweep and projection loop indefinitely.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_planar_display(screen, projector, frames, sweep_frequency):
    """
    Renders a 3D volume by projecting 2D frames onto a moving screen.
    
    Args:
        screen: Object with methods move_to_z(z) and get_z()
        projector: Object with method project(image)
        frames: List of N 2D image arrays
        sweep_frequency: Frequency of screen oscillation in Hz (> 24)
    """
    cycle_duration = 1.0 / sweep_frequency
    num_frames = len(frames)
    
    # Assume linear sweep back and forth for simplicity
    # In practice, trajectory may be sinusoidal or triangular
    # The key is that the screen covers the Z-range continuously
    
    while True:
        start_time = current_time()
        
        # Forward sweep
        for i in range(num_frames):
            # Map frame index to Z position
            # Assuming uniform distribution across the volume depth
            z_pos = map_z(i, num_frames)
            
            # Move screen to z_pos
            # Note: In a real system, this is continuous motion.
            # The projector must trigger exactly when the screen is at z_pos
            screen.move_to_z(z_pos)
            
            # Project the frame
            projector.project(frames[i])
            
            # Wait for next frame timing to maintain synchronization with motion
            # The time between projections must match the screen velocity
            wait_for_next_frame_projection()
            
        # Backward sweep (mirror the frames or use separate backward frames)
        for i in range(num_frames - 1, -1, -1):
            z_pos = map_z(i, num_frames)
            screen.move_to_z(z_pos)
            projector.project(frames[i])
            wait_for_next_frame_projection()

def map_z(frame_index, total_frames):
    """
    Maps a frame index to a Z-coordinate.
    Implementation detail [G]: Assumes linear Z-spacing.
    """
    # Normalize index to [0, 1]
    t = frame_index / (total_frames - 1)
    # Map to physical Z range [Z_min, Z_max]
    return Z_min + t * (Z_max - Z_min)
```

**Edge Cases and Pitfalls:**
*   **Flicker:** If $f_{sweep} \le 24$ Hz, temporal integration fails, causing visible flicker [S1].
*   **Motion Blur:** If the screen velocity is too high relative to the projector's frame rate, images may blur. High-speed DLP engines are recommended to mitigate this [S1].
*   **Synchronization:** Precise timing between screen position and image projection is critical. Misalignment results in distorted Z-depths.

## Subtopics

### [Line Patterns and 2D/3D Projection](10357-line-patterns-and-2d-3d-projection.md)
A signal modulation technique that projects variable 1D patterns along the viewing line through the display volume by timing laser pulses. It provides depth context without the occlusion of full 3D shapes, serving as the 3D analog to the 0D dot used on 2D screens.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of frames $N$ per sweep cycle. The frame rate is constrained by the mechanical limits of the screen actuator and the projector's speed.
*   **Memory Complexity:** $O(N \times W \times H)$, where $W$ and $H$ are the resolution of the 2D projector.
*   **Performance:** Relies on high-speed DLP projection engines to achieve sufficient frame rates for temporal integration [S1].
*   **Strengths:** Provides true 3D spatial representation with omni-directional light emission, offering both physiological and psychological depth cues [S1]. Simplifies understanding of 3D spatial relationships [S1].
*   **Limitations:** Mechanical complexity of the moving screen. Resolution is limited by the projector's resolution and the screen's surface properties. Requires precise synchronization.
*   **When to Use:** Applications requiring true 3D visualization without glasses, such as medical imaging or scientific visualization where depth perception is critical.
*   **When Not to Use:** When high resolution or large volume size is required without mechanical complexity, or when power consumption of high-speed actuators is a concern.

## Background
The multi-planar volumetric 3D display technique was developed to address the limitations of conventional 3D rendering on 2D flat screens, which lack physical depth cues [S1]. The method leverages high-speed DLP (Digital Light Processing) technology to facilitate the high-speed image projection necessary for volumetric display [S1]. The approach was detailed in a study published in 2012, which designed a volumetric 3D display based on a DLP projection engine and developed a high-speed projection engine to facilitate the system [S1].

## Sources
[S1] A volumetric 3D display based on a DLP projection engine, https://doi.org/10.1016/j.displa.2012.11.001, fetched 2026-10-27
