# Seelinder

Also called: Cylindrical 3D display with parallax barrier, Cylindrical Parallax Barrier Display, Rotating LED array volumetric display with parallax barrier

## Overview
Seelinder is a 360° autostereoscopic volumetric display that generates a horizontal parallax barrier (HPO) viewing area using a rotating cylindrical parallax barrier and a 1D LED array [S1, unofficial][S2, unofficial]. It solves the physical limitation of mounting a full 360° array of LED sources by rotating a single linear array synchronously with a slotted cylindrical barrier [S1, unofficial]. Unlike static parallax barrier displays, Seelinder creates multiple viewpoints in 360 degrees by scanning light rays as the slit and LED positions change relative to each other [S2, unofficial]. It builds on [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md) by using the barrier itself as the spatial multiplexing element rather than a continuous emissive screen.

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of 2D view images corresponding to different horizontal viewing angles; rotation speed $\omega$; barrier geometry parameters.
*   **Output:** A 3D volumetric image visible from any position surrounding the device (outward viewing) or inside it (inward viewing).

**Data Structures:**
The hardware consists of two primary rotating components:
1.  **Cylindrical Parallax Barrier:** A cylinder with vertical slits.
    *   **Barrier Interval ($B$):** The center-to-center distance between adjacent slits.
    *   **Aperture Width ($A$):** The width of the open slit.
    *   **Height ($H$):** Vertical extent of the barrier.
2.  **LED Array:** A 1D linear array of LEDs.
    *   **LED Width ($W_{LED}$):** Physical width of each LED emitter.
    *   **Position:** Located at a fixed radial distance $D$ from the cylinder axis, inside the barrier (for outward viewing) or outside (for inward viewing).

**Numbered Steps:**
1.  **Synchronization Setup:** The LED array and the cylindrical parallax barrier rotate together at a constant angular velocity $\omega$. In the standard "outward viewing" configuration, the barrier is outer and LEDs are inner [S1, unofficial]. In the "inward viewing" variant, the viewer is inside the cylinder, barriers are inner, LEDs are outer, and both rotate together [S1, unofficial].
2.  **Viewpoint Mapping:** The 360° horizontal field of view is divided into $N$ discrete viewing zones (views). For a prototype, $N > 70$ [S1, unofficial]. Each zone corresponds to a specific angular interval $\Delta \theta < 1^\circ$ [S1, unofficial].
3.  **Image Generation:** For each rotation angle $\theta(t)$, the system determines which LED in the 1D array is aligned with which slit in the barrier. The LED array displays the 2D image slice corresponding to the viewer's current angular position relative to the barrier slits.
4.  **Light Ray Scanning:** As the assembly rotates, the relative motion between the slits and the LEDs scans light rays across the horizontal plane. This scanning reproduces multiple viewpoints in 360 degrees [S2, unofficial].
5.  **Temporal Integration:** The persistence of vision blends the discrete 2D slices projected through the moving slits into a continuous 3D volume [G]. The refresh rate must be sufficient to maintain image stability.

**Formulas:**
*   **Angular Parallax Interval:** $\Delta \theta = \frac{360^\circ}{N}$, where $N$ is the number of views.
*   **Pixel Viewing Angle:** The source specifies a pixel viewing angle of $60^\circ$ divided into over 70 views [S1, unofficial]. This implies the optical design ensures each LED pixel contributes to a specific angular sector.
*   **Synchronization Condition:** The frame update rate $f_{frame}$ and rotation speed $\omega$ must satisfy the condition that the image content updates in phase with the barrier slits passing the LEDs. If the barrier has $M$ slits, the system may update frames or sub-images as slits align with LEDs.

**Parameters (Prototype Values):**
*   **Diameter:** 200 mm [S1, unofficial].
*   **Height:** 256 mm [S1, unofficial].
*   **Resolution:** 1254 circumferential pixels, 256 vertical pixels [S1, unofficial].
*   **Refresh Rate:** 30 Hz [S1, unofficial].
*   **Views:** Over 70 views [S1, unofficial].
*   **Key Design Parameters:** Barrier interval ($B$), aperture width ($A$), LED width ($W_{LED}$), and distance between LEDs and barriers ($D$) [S1, unofficial].

**Pitfalls:**
*   **Misalignment:** Precise timing between the screen's angular position and the image update is critical. Misalignment results in Z-depth distortion [G].
*   **Physical Limits:** The number of LEDs is physically limited by the 1D array size; rotation is required to cover 360° [S1, unofficial].
*   **Viewing Zone Accuracy:** Each viewing location (zone) must receive the appropriate view corresponding to the viewer's location. Incorrect mapping causes crosstalk or incorrect depth perception [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Seelinder Control Logic
# Constants from prototype
Diameter_mm = 200
Height_mm = 256
Circumferential_Pixels = 1254
Vertical_Pixels = 256
Refresh_Hz = 30
Num_Views = 70  # "over 70"
Pixel_Viewing_Angle_deg = 60

# Hardware state
rotation_angle_deg = 0.0
angular_velocity_deg_per_sec = 360.0 * Refresh_Hz  # Example: 1 rotation per refresh cycle? 
# Note: Exact relationship between rotation speed and frame rate is not explicitly defined in sources.
# Assumption: The system rotates continuously, and LEDs update at Refresh_Hz.

def get_led_intensity(led_index, time_step):
    """
    Determines the intensity of a specific LED in the 1D array.
    led_index: Index in the 1D LED array (0 to Num_LEDs-1).
    time_step: Current time index.
    """
    # Calculate current rotation angle
    current_angle = (rotation_angle_deg + angular_velocity_deg_per_sec * time_step) % 360.0
    
    # Determine which viewing zone the current LED position corresponds to
    # This depends on the barrier slit position relative to the LED
    # In a rotating system, the relative position changes continuously.
    # The LED array must display the view corresponding to the viewer's angle.
    # Viewer angle is fixed in world space. LED angle is rotating.
    # The slit acts as a window.
    
    # Simplified model: The LED array displays a panoramic image.
    # The rotation maps the panoramic image to the correct viewing angle.
    viewer_angle = get_viewer_position() # External input
    
    # Map viewer angle to a view index
    view_index = int(viewer_angle / (360.0 / Num_Views)) % Num_Views
    
    # The LED at the current angular position should emit the pixel for this view
    # The specific mapping between LED index and angular position depends on the array geometry
    # and the barrier slit pattern.
    
    # Placeholder for complex optical mapping logic not fully detailed in sources
    # The source states: "In each viewing location (zone), display an appropriate view"
    # and "utilizes the scanning of light rays when slit and LED positions change"
    
    return get_view_pixel(view_index, led_index)

def update_display():
    global rotation_angle_deg
    while True:
        # Rotate the assembly
        rotation_angle_deg += angular_velocity_deg_per_sec / Refresh_Hz
        
        # Update LEDs based on current rotation and viewer positions
        for led_idx in range(Num_LEDs):
            intensity = get_led_intensity(led_idx, 0)
            set_led(led_idx, intensity)
            
        wait_for_refresh()
```

## Subtopics

### Inward Viewing Seelinder
A variant where the viewer is inside a large-size cylinder, barriers are inside, and LEDs are outside. Both rotate together [S1, unofficial]. Chosen when the display needs to be a central object surrounded by viewers, or for immersive cabin-like experiences.

### Outward Viewing Seelinder
The standard design where viewers surround the outside of the device. Barriers are outer, LEDs are inner [S1, unofficial]. Chosen for public displays or interactive installations where viewers walk around the device.

## Operating characteristics
*   **Time Complexity:** Real-time rendering required. Frame rate is 30 Hz [S1, unofficial].
*   **Memory Complexity:** Resolution is 1254 x 256 pixels [S1, unofficial].
*   **Performance:** Prototype dimensions are 200 mm diameter, 256 mm height [S1, unofficial].
*   **Strengths:** Provides 360° autostereoscopic viewing without glasses; overcomes the physical limitation of mounting a full 360° LED array by using rotation [S1, unofficial].
*   **Limitations:** Limited by the number of views (over 70) and angular parallax interval (< 1°) [S1, unofficial]. Requires precise synchronization between rotation and image update [G].
*   **When to use:** When a 360° autostereoscopic display is needed for a small-scale prototype or specific installation.
*   **When not to use:** When high resolution or large scale is required, as the rotating mechanism adds complexity and potential mechanical failure points.

## Background
Seelinder was proposed by Endo et al. in 2010 [S1, unofficial][S2, unofficial]. It is an example of extending the field of view of autostereoscopic displays to 360° using a rotating cylindrical parallax barrier and LED arrays [S1, unofficial]. The design is part of the broader category of free-space volumetric displays that do not require a physical medium like dust or gas [S2, unofficial].

## Sources
[S1] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
