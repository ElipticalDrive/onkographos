# Sketch-based Transfer Function Editing

Also called: Voxel TIPs, Free-Form Sketching, Multi-touch Transfer Function Editing, direct control approach, sketch-based interface

## Overview
Sketch-based Transfer Function Editing is a method for designing volume rendering transfer functions using direct manipulation gestures on an interactive surface, specifically implemented in the **Voxel TIPs** system [S1, unofficial]. It solves the difficulty and tedium of designing opacity and color mappings with conventional WIMP (Mouse/Keyboard) interfaces by allowing users to sketch, scan, and refine intensity- and gradient-based transfer functions using multi-touch gestures [S1, unofficial]. It sets itself apart from siblings like [Position-dependent Transfer Function Customization](5287-position-dependent-transfer-function-customization.md) by focusing on global domain manipulation via sliding windows and mixing tables rather than spatial position, and from [Automated Classification Widget Placement](5645-automated-classification-Widget-placement.md) by relying on explicit user sketching rather than automated widget placement [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User touch coordinates on a multi-touch surface; scalar volume data (intensity $v$ and gradient magnitude $\|\nabla f\|$).
*   **Output:** A 1D transfer function mapping scalar values to opacity $\alpha \in [0, 1]$ and color $C(r, g, b)$, and a rendered volume image.

**Data Structures:**
The interface maintains the following state:
1.  **Transfer Function (TF):** A piecewise function defined by control points (nodes) and segments.
    *   *Ramp-like:* Linear interpolation between nodes.
    *   *Sliding Window:* Rectangular pulse defined by center, width, and amplitude.
    *   *Mixing Table:* Array of 10 vertical sliders, each associated with a fixed domain interval (10% of full domain) and an adjustable height (opacity).
    *   *Sketch:* A set of points $(x_i, y_i)$ defining the curve.
2.  **Color Map:** A list of color handles with HSV values.
3.  **Thumbnails:** Visual representations of data subsets for each 10% domain interval, rendered as particle-based volume rendering [S1, unofficial].

**Numbered Steps:**

1.  **Initialize Interface:**
    *   Display three areas: Upper (volume rendering), Bottom-Left/Right (TF editing for intensity/gradient), and Bottom (thumbnails) [S1, unofficial].
    *   Initialize TF to full opacity (all terms at full opacity) [S1, unofficial].

2.  **Geometric Transformation (Upper Area):**
    *   Apply touch gestures to rotate, translate, or scale the volume rendering:
        *   One tap and drag: Angle-center rotation.
        *   Two taps and press/revolve: Rotation along axis orthogonal to screen.
        *   Pinch or two-finger drag: Uniform scale.
        *   Multi-touch drag: Translation.
        *   Three+ consecutive taps: Reset position/orientation [S1, unofficial].

3.  **Select Editing Mode:**
    *   Choose one of four modes for the active domain (Intensity or Gradient):
        *   **Ramp-like:** Piecewise linear function.
        *   **Sliding Window:** Rectangular pulse.
        *   **Mixing Table:** 10 vertical sliders.
        *   **Sketching:** Free-form drawing [S1, unofficial].

4.  **Edit Transfer Function (Lower Area):**
    *   **If Ramp-like:** Vertically move linear interpolation nodes. Intermediate values are determined by linear interpolation between nodes [S1, unofficial].
    *   **If Sliding Window:**
        *   Drag two cursors at top edges to adjust pulse width and amplitude.
        *   Drag window through domain to scan and highlight intervals. Opacity is constant within interval, zero outside [S1, unofficial].
    *   **If Mixing Table:**
        *   Drag vertical sliders to adjust height (opacity) for each 10% domain segment.
        *   Tap thumbnails to toggle opacity of the entire segment to 0 or 1. Enables selection of multiple areas of interest [S1, unofficial].
    *   **If Sketching:**
        *   Tap and drag to draw 1D graph strokes.
        *   Each stroke updates the function at selected domain values. Allows fine-tuning and intricate details [S1, unofficial].

5.  **Assign Colors:**
    *   Double-tap color bar to evoke a handle and HSV color picker.
    *   Select color and apply to handle.
    *   Drag handle to change position on color bar.
    *   Drag handle outside color bar to delete.
    *   Tapping handle opens/closes picker. Color interpolation occurs between neighboring handles [S1, unofficial].

6.  **Tool Switching Adaptation:**
    *   When switching modes, the prior TF is adapted to fit the new tool type:
        *   Ramp-like $\to$ Sketching: TF unchanged.
        *   Sketching $\to$ Mixing Table: TF converted to rectangular pulse train based on average values in each slider's range [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelTIPsInterface:
    def __init__(self, volume_data):
        self.volume = volume_data
        self.tf_intensity = RampTF()
        self.tf_gradient = RampTF()
        self.color_handles = []
        self.thumbnails = [Thumbnail(i) for i in range(10)] # 10 segments of 10% domain
        self.mode_intensity = "ramp"
        self.mode_gradient = "ramp"
        self.view_transform = IdentityMatrix()

    def handle_geometric_gesture(self, gesture_type, coordinates):
        if gesture_type == "tap_drag":
            self.view_transform = Rotate(self.view_transform, angle=calculate_angle(coordinates))
        elif gesture_type == "two_tap_revolve":
            self.view_transform = RotateAxis(self.view_transform, axis=orthogonal_to_screen, angle=calculate_revolve(coordinates))
        elif gesture_type == "pinch":
            self.view_transform = Scale(self.view_transform, factor=calculate_scale(coordinates))
        elif gesture_type == "multi_drag":
            self.view_transform = Translate(self.view_transform, delta=calculate_delta(coordinates))
        elif gesture_type == "triple_tap":
            self.view_transform = IdentityMatrix()

    def handle_tf_gesture(self, domain, gesture_type, coordinates):
        if domain == "intensity":
            self._apply_gesture_to_tf(self.tf_intensity, self.mode_intensity, gesture_type, coordinates)
        else:
            self._apply_gesture_to_tf(self.tf_gradient, self.mode_gradient, gesture_type, coordinates)

    def _apply_gesture_to_tf(self, tf, mode, gesture_type, coordinates):
        if mode == "ramp":
            if gesture_type == "node_move":
                tf.move_node(coordinates.x, coordinates.y)
        elif mode == "sliding_window":
            if gesture_type == "cursor_drag":
                tf.adjust_pulse(coordinates.x, coordinates.y) # Adjust width/amplitude
            elif gesture_type == "window_drag":
                tf.translate_window(coordinates.x)
        elif mode == "mixing_table":
            if gesture_type == "slider_drag":
                tf.set_slider_height(coordinates.x, coordinates.y)
            elif gesture_type == "thumbnail_tap":
                tf.toggle_segment_opacity(coordinates.x) # x is segment index
        elif mode == "sketching":
            if gesture_type == "draw_stroke":
                tf.add_stroke(coordinates)

    def handle_color_gesture(self, gesture_type, coordinates):
        if gesture_type == "double_tap":
            handle = ColorHandle(coordinates.x)
            handle.open_picker()
            self.color_handles.append(handle)
        elif gesture_type == "handle_drag":
            for h in self.color_handles:
                if h.contains(coordinates):
                    h.position = coordinates.x
        elif gesture_type == "handle_delete":
            if not self.color_bar.contains(coordinates):
                self.color_handles.remove(coordinates.handle)

    def switch_mode(self, domain, new_mode):
        old_mode = self.mode_intensity if domain == "intensity" else self.mode_gradient
        tf = self.tf_intensity if domain == "intensity" else self.tf_gradient
        
        if old_mode == "sketching" and new_mode == "mixing_table":
            tf.convert_to_rectangular_pulse_train()
        # Other transitions leave TF unchanged or adapt as per source
        
        if domain == "intensity":
            self.mode_intensity = new_mode
        else:
            self.mode_gradient = new_mode

    def render(self):
        # Apply view_transform to volume
        # For each ray, sample volume
        # For each sample, get intensity v and gradient g
        # alpha = max(self.tf_intensity(v), self.tf_gradient(g))
        # color = interpolate_color(self.color_handles, v)
        # Accumulate using ray marching
        pass
```

**Parameters:**
*   **Thumbnail Interval Size:** 10% of full parameter domain [S1, unofficial].
*   **Mixing Table Sliders:** 10 equally sized sliders [S1, unofficial].
*   **Test Volume Resolution:** Not given in the sources.
*   **Hardware:** Samsung UE55F8000 55” TV, ZaagTech X Series 16 Touch Overlay [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Tool Switching:** Switching from sketching to mixing table converts the function to a rectangular pulse train based on average values. This may lose fine details from the sketch [S1, unofficial].
*   **Precision:** Touch gestures on a large tabletop may have lower precision than mouse input, but users found them more accurate for reproducing intentions [S1, unofficial].

## Subtopics

### [Interactive Volume Rendering with Multi-Dimensional TFs](6183-interactive-volume-rendering-with-multi-dimensional-tfs.md)
An interactive method using 3D transfer functions (value, 1st, 2nd derivative) where users can make regions opaque by pointing at them and 'paint' into the transfer function domain. It differs from Sketch-based TF Editing by operating in a higher-dimensional domain (3D vs 1D) and using a 'painting' metaphor rather than sliding windows or mixing tables. Choose this when complex multi-feature classification is needed. [6183-interactive-volume-rendering-with-multi-dimensional-tfs.md]

### Sliding Window Scanning
A technique to scan the intensity and gradient magnitude domains to sketch initial concepts for intensity-based and gradient-based transfer function terms. It is a sub-tool within Sketch-based TF Editing (Voxel TIPs) used to highlight selected domain intervals with a rectangular pulse. It is chosen for rapid initial approximation of the TF domain. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In user tests with 15 participants, Voxel TIPs was compared to VoTracer (WIMP/mouse). Completion times for intensity and gradient TF design were generally faster or comparable with Voxel TIPs, with users preferring touch gestures for accuracy [S1, unofficial].
*   **Strengths:** Allows rapid initial approximation via sliding windows and mixing tables; fine-tuning via free-form sketching; supports both intensity and gradient TFs; bimanual interaction; no complex mathematical formulations required [S1, unofficial].
*   **Limitations:** Requires specialized hardware (multi-touch tabletop); limited to 1D TFs in the base implementation; precision may vary with touch input [S1, unofficial].
*   **When to Use:** When rapid, intuitive design of 1D transfer functions is needed, and multi-touch hardware is available.
*   **When Not to Use:** When high-precision mouse input is required, or when multi-dimensional TFs are necessary.

## Background
The method was introduced by D. S. Lopes et al. in 2017 as "Voxel TIPs" (Voxel Transfer Function Interactive Platform) [S1, unofficial]. It was developed to address the difficulties of designing transfer functions using conventional WIMP interfaces. The system was tested on medical CT datasets (pelvis, brain, lung) and compared against VoTracer. It builds on the principle of direct control and sketch-based interaction [S1, unofficial].

## Sources
[S1] Explicit design of transfer functions for volume-rendered images, https://web.tecnico.ulisboa.pt/daniel.s.lopes/papers/Explicit%20design%20of%20transfer%20functions%20for%20volume-rendered%20images%20-%20Vis%20Comp%202017.pdf, fetched 2026-10-29 (unofficial)
