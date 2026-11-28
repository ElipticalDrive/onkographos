# Omniview View Volume Control

Also called: Direct Manipulation via Joystick, Tracking Window

## Overview
Omniview View Volume Control is a user interface technique for a true 3-D display device that allows users to control the position and size (radius) of the 3D view volume using a mouse tracking window and a joystick [S1, unofficial]. It provides a direct manipulation alternative to the indirect WIMP (Windows, Icons, Menus, Pointer) interface, enabling rapid large-scale movements to a desired display area [S1, unofficial]. While the WIMP interface offers finer control, it requires the user to be 3-4 feet from the display, whereas the direct manipulation methods allow for more immersive interaction [S1, unofficial].

## How it works
The method combines two input modalities: a 2D mouse-based tracking window for spatial orientation and a joystick for precise control of the view volume's location, scale, and data value.

### Inputs and Outputs
*   **Inputs:** Mouse (for tracking window), Joystick (stick movement, forward/backward tilt, top button, bottom button).
*   **Outputs:** Updated 3D view volume parameters (center position, radius, displayed value range) rendered on the Omniview display.

### Data Structures
The tracking window is a 2D map representing the area of interest, similar to a topographic or road map [S1, unofficial].
*   **Tracking Map:** A 2D coordinate system representing the global area of interest.
*   **View Volume Indicator:** A circle rendered on the tracking map.
    *   **Center:** Represents the center of the Omniview display.
    *   **Radius:** Represents the area being viewed.

### Parameters
*   **View Volume Radius:** The radius of the view volume. In the described battlefield environment, this is typically greater than 2,500 meters and up to 500,000 meters [S1, unofficial].
*   **Joystick State:**
    *   `stick_x`, `stick_y`: Horizontal and vertical displacement of the joystick stick.
    *   `tilt_north`: Boolean, true if joystick is tilted forward.
    *   `tilt_south`: Boolean, true if joystick is tilted backward.
    *   `btn_top`: Boolean, true if top button is pressed.
    *   `btn_bottom`: Boolean, true if bottom button is pressed.

### Steps
1.  **Initialize Tracking Window:** Display a 2D map of the area of interest. Render a circle on this map representing the current center and radius of the Omniview view volume [S1, unofficial].
2.  **Mouse Control (Position):**
    *   Detect user drag of the circle on the tracking map.
    *   Update the center position of the view volume based on the drag delta.
    *   Update the circle's position on the tracking map to reflect the new center [S1, unofficial].
3.  **Joystick Position Control:**
    *   Map joystick stick movement (horizontal/vertical displacement) to changes in the location of the view volume center [S1, unofficial].
4.  **Joystick Scale Control:**
    *   If neither button is pressed:
        *   If `tilt_north` is true, increase the view volume radius.
        *   If `tilt_south` is true, decrease the view volume radius [S1, unofficial].
5.  **Joystick Value Control:**
    *   If `btn_top` is pressed:
        *   If `tilt_north` is true, raise the value being viewed.
        *   If `tilt_south` is true, lower the value being viewed [S1, unofficial].
    *   If `btn_bottom` is pressed:
        *   If `tilt_north` is true, raise the value being viewed.
        *   If `tilt_south` is true, lower the value being viewed [S1, unofficial].
6.  **Feedback Update:** Update the tracking map to show the change in location and size of the volume being displayed [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OmniviewViewVolumeControl:
    def __init__(self, initial_radius=2500):
        # View volume state
        self.view_center = (0, 0)
        self.view_radius = initial_radius
        self.viewed_value = 0.0
        
        # Tracking window state
        self.tracking_map_center = self.view_center
        self.tracking_map_radius = self.view_radius
        
        # Joystick state
        self.joystick_stick = (0, 0)
        self.joystick_tilt_north = False
        self.joystick_tilt_south = False
        self.joystick_btn_top = False
        self.joystick_btn_bottom = False
        
        # Mouse state
        self.mouse_dragging = False
        self.mouse_drag_start = (0, 0)
        self.mouse_drag_current = (0, 0)

    def update_mouse(self, event):
        """Handle mouse events for tracking window interaction."""
        if event.type == "drag_start":
            self.mouse_dragging = True
            self.mouse_drag_start = event.position
            self.mouse_drag_current = event.position
        elif event.type == "drag_move":
            if self.mouse_dragging:
                self.mouse_drag_current = event.position
                # Update view center based on drag delta
                delta = (
                    self.mouse_drag_current[0] - self.mouse_drag_start[0],
                    self.mouse_drag_current[1] - self.mouse_drag_start[1]
                )
                self.view_center = (
                    self.view_center[0] + delta[0],
                    self.view_center[1] + delta[1]
                )
                self.tracking_map_center = self.view_center
                self.mouse_drag_start = self.mouse_drag_current
        elif event.type == "drag_end":
            self.mouse_dragging = False

    def update_joystick(self, state):
        """Handle joystick input for position, scale, and value control."""
        self.joystick_stick = state.stick
        self.joystick_tilt_north = state.tilt_north
        self.joystick_tilt_south = state.tilt_south
        self.joystick_btn_top = state.btn_top
        self.joystick_btn_bottom = state.btn_bottom

        # Position Control: Map stick movement to location change
        # Note: Exact mapping factor not specified in sources [G]
        position_scale = 1.0
        self.view_center = (
            self.view_center[0] + self.joystick_stick[0] * position_scale,
            self.view_center[1] + self.joystick_stick[1] * position_scale
        )
        self.tracking_map_center = self.view_center

        # Scale Control: Map tilt to radius change
        if not self.joystick_btn_top and not self.joystick_btn_bottom:
            scale_step = 100.0 # [G] Placeholder step size
            if self.joystick_tilt_north:
                self.view_radius += scale_step
            elif self.joystick_tilt_south:
                self.view_radius -= scale_step
            # Clamp radius if necessary [G]
            self.view_radius = max(0, self.view_radius)
            self.tracking_map_radius = self.view_radius

        # Value Control: Map tilt + button to value change
        value_step = 1.0 # [G] Placeholder step size
        if self.joystick_btn_top or self.joystick_btn_bottom:
            if self.joystick_tilt_north:
                self.viewed_value += value_step
            elif self.joystick_tilt_south:
                self.viewed_value -= value_step

    def render_tracking_window(self):
        """Render the 2D map and view volume indicator."""
        # Draw 2D map of area of interest
        draw_2d_map()
        # Draw circle representing view volume center and radius
        draw_circle(self.tracking_map_center, self.tracking_map_radius)

    def render_omniview(self):
        """Render the 3D view volume on the Omniview display."""
        # Render 3D scene with current view center, radius, and value
        render_3d_scene(self.view_center, self.view_radius, self.viewed_value)
```

### Edge Cases and Pitfalls
*   **Precision vs. Scale:** For increasingly large view volumes (up to 500,000 meters), accurate scaling of objects may cause them to become smaller than a single voxel, making them invisible [S1, unofficial]. The system prioritizes relative positioning and spacing over precise detail [S1, unofficial].
*   **User Distance:** The direct manipulation interface (joystick/voice) does not provide all controls that the indirect WIMP interface does. Users relying solely on direct manipulation may lack fine-grained control [S1, unofficial].
*   **Button Ambiguity:** Both the top and bottom buttons perform the same function (raising/lowering value based on tilt direction). The distinction between top and bottom buttons is not explained in the sources [S1, unofficial].

## Subtopics

### Interactive Editing with Proxy Plane Manipulation
Uses proxy planes for editing volume data. Chosen when precise local editing is needed. [Interactive Editing with Proxy Plane Manipulation](19238-interactive-editing-with-proxy-plane-manipulation.md)

### Focus Marker with Smooth Interpolation
Uses a focus marker for smooth transitions. Chosen when smooth visual transitions are prioritized over direct manipulation. [Focus Marker with Smooth Interpolation](22729-focus-marker-with-smooth-interpolation.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Allows rapid large-scale movements to the desired display area. Provides direct manipulation which can be more intuitive for spatial tasks.
*   **Limitations:** Does not provide all controls available in the indirect WIMP interface. Requires user to be close to the display for effective joystick use, whereas WIMP allows for distance.
*   **When to Use:** When rapid navigation and direct spatial control are prioritized over fine-grained parameter adjustment.
*   **When Not to Use:** When fine-grained control is required, or when the user needs to be at a distance from the display.

## Background
The Omniview View Volume Control method was developed for the SIMNET battlefield simulation environment. It was part of a user interface designed for a true 3-D display device, aiming to provide effective visualization of battlefield data. The system was described in a report from the Defense Technical Information Center (DTIC) [S1, unofficial].

## Sources
[S1] DTIC ADA258995: A User Interface to a True 3-D Display Device, https://archive.org/details/DTIC_ADA258995, fetched 2026-10-26 (unofficial)
