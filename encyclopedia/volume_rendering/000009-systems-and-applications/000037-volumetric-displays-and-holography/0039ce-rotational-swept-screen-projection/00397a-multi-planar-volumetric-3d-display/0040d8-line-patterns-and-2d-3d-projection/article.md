# Line Patterns and 2D/3D Projection

Also called: Line Pattern Volumetric Display, Temporal Laser Line Projection

## Overview
A signal modulation technique that projects variable 1D patterns along the viewing line through the display volume by timing laser pulses [S1]. It provides depth context without the occlusion of full 3D shapes, serving as the 3D analog to the 0D dot used on 2D screens [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Time-varying control signals (e.g., from a joystick or distance sensor) and the display's oscillation frequency.
*   **Output:** A modulated laser beam forming 1D patterns (points, lines, dashed segments) within the volumetric display space.

**Data Structures:**
*   **Control Signal:** A time-indexed sequence of duty cycle or pulse-width values.
*   **Oscillation State:** The current phase of the display screen's rotation/oscillation.

**Steps:**
1.  **Synchronization:** Align the laser control signal with the display's oscillation frequency. The phase of the signal relative to the oscillation determines the depth (Z-axis) where the pattern appears [S1].
2.  **Pattern Generation:** Modulate the laser's pulse width or duty cycle over time to create specific 1D shapes:
    *   **Single Point:** A short pulse duration.
    *   **Continuous Line:** A sustained pulse duration covering the desired length.
    *   **Dashed Line/Segments:** Alternating on/off states within the pulse window.
3.  **Depth Control:** Adjust the phase offset of the control signal to move the pattern along the Z-axis [S1].
4.  **Animation:** Play temporal animations (e.g., moving segments) to encode direction or speed along the line [S1].

**Formulas:**
*   **Drift Error:** The positional drift of the laser dot is a function of the frequency mismatch between the laser controller and the display.
    $$ \text{Drift} = \Delta f \times t $$
    Where $\Delta f$ is the frequency difference and $t$ is time. In a Voxon VX1 display with an Arduino Nano (16 MHz Timer 1), a default 30 Hz oscillation results in a drift of only 5 mm per minute [S1].

**Parameters:**
*   **Oscillation Frequency:** Typically 30 Hz [S1].
*   **Controller Resolution:** 16 MHz timer resolution (e.g., Arduino Nano) [S1].
*   **Interaction Distance:** 30 cm to 2 meters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_laser_signal(oscillation_phase, pattern_type, length, dash_ratio):
    """
    Generates the duty cycle for the laser diode based on the current
    oscillation phase and desired pattern.
    
    Args:
        oscillation_phase: Float [0, 1], representing the current position 
                           of the screen in its oscillation cycle.
        pattern_type: Enum {POINT, LINE, DASHED}
        length: Float, relative length of the line segment.
        dash_ratio: Float [0, 1], ratio of 'on' time to total period for dashed lines.
    
    Returns:
        Float: Duty cycle (0.0 to 1.0) for the current sample.
    """
    # Determine the 'on' window for the pattern relative to the phase
    # The phase maps to the Z-depth. The pattern extends along the line 
    # passing through the volume at that depth.
    
    if pattern_type == POINT:
        # Short pulse centered at the target phase
        pulse_width = 0.05 # Arbitrary small width for a point
        center = oscillation_phase
        if abs(oscillation_phase - center) < pulse_width / 2:
            return 1.0
        else:
            return 0.0
            
    elif pattern_type == LINE:
        # Continuous line of specified length
        # The line extends along the viewing line through the volume.
        # In practice, this is a sustained pulse while the screen passes 
        # through the relevant Z-range.
        return 1.0 # Simplified: Laser is on for the duration of the line's Z-span
        
    elif pattern_type == DASHED:
        # Dashed line with a dot at the pointing position
        # Modulate duty cycle to create dashes
        cycle = (oscillation_phase * 10) % 1.0 # Create repeating cycle
        if cycle < dash_ratio:
            return 1.0
        else:
            return 0.0
            
    return 0.0

def synchronize_laser(display_freq, controller_freq):
    """
    Adjusts the controller to minimize drift.
    """
    # Frequency needs to be adjusted only once per display [S1]
    # Default 30 Hz oscillation with 16 MHz timer resolution 
    # yields 5mm/min drift [S1]
    pass
```

**Edge Cases:**
*   **Hand Jittering:** Hand jitter is less noticeable at distances of 30 cm to 2 meters [S1].
*   **Arm Fatigue:** A disadvantage inherited from traditional laser pointers [S1].

## Subtopics

### Complex Timing Patterns (Dashed line + dot)
A visual pattern modification where the laser output is pulsed to create a dashed line with a solid dot at the pointing position, rather than a continuous line or single point [S1]. The laser is pulsed at the display's oscillation frequency, but the duty cycle or timing is modulated to produce the dashed appearance, with the solid dot appearing at the specific phase/height selected by the user [S1]. This pattern was employed in depth user studies [S1].

## Operating characteristics
*   **Strengths:** Provides depth context without occlusion; allows for natural interaction (e.g., pinching, rotation) using wearable devices [S1].
*   **Limitations:** Hand jitter and arm fatigue are disadvantages [S1]. Higher-dimensional pointers (spheres, cubes) are considered beyond the scope of current laser pointer implementations [S1].
*   **Performance:** Drift is minimal (5 mm/min) with standard hardware (Arduino Nano, 30 Hz display) [S1].

## Background
Originated with the PointerVol device, a laser pointer designed for swept volumetric displays [S1]. The concept draws an analogy between 2D screens (using 0D dots) and 3D volumes (using 1D line patterns) [S1]. Future research directions include higher-dimensional pointers (spheres, cubes, arrows) and independent control of multiple pointers for complex 3D rotations [S1].

## Sources
[S1] PointerVol: A Laser Pointer for Swept Volumetric Displays, https://doi.org/10.1145/3654777.3676432, fetched 2026-10-30
