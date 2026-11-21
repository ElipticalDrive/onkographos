# Sketch-based Transfer Function Editing

Also called: Sketch-based TF Design, Sketch-based TF Design using Strokes, Sketch-based 1D Transfer Function Design (Strokes), Sketch-based Transfer Function Design (Slices)

## Overview
Sketch-based Transfer Function (TF) Editing allows users to define or modify classification functions by drawing strokes directly on the volume rendering view or on dedicated 1D/2D parameter spaces, rather than manipulating abstract widgets [S1, unofficial][S2, unofficial]. Unlike [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) which optimizes widget positions, or [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) which uses geometric handles on a 2D plot, sketch-based methods prioritize direct manipulation and intuitive mapping between visual intent and parameter space [S2, unofficial]. It solves the difficulty of manually tuning numerical parameters by enabling rapid, global definition of TF shapes and local refinement through free-form drawing [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User-drawn strokes (2D coordinates in screen space or parameter space), initial volume data (scalar field $f(\mathbf{x})$ and optionally gradient magnitude $\|\nabla f(\mathbf{x})\|$), and current TF state [S1, unofficial][S2, unofficial].
*   **Output:** A modified Transfer Function mapping scalar/gradient values to opacity $\alpha$ and color $C$ [S1, unofficial][S2, unofficial].

**Data Structures:**
The TF is typically represented as a piecewise linear function defined by keypoints (nodes) in the parameter domain (intensity or gradient magnitude) [S2, unofficial]. In systems like Voxel TIPs, the domain is discretized into segments for specific tools (e.g., 10 equal segments for sliders) [S2, unofficial].

**Steps:**
1.  **Sketch Acquisition:** The user draws strokes on the interface [S1, unofficial][S2, unofficial].
    *   *Spatial Sketching:* Strokes are drawn on the rendered volume image [S1, unofficial].
    *   *Parameter Sketching:* Strokes are drawn on a 1D graph (intensity/gradient axis) or 2D histogram [S2, unofficial].
2.  **Interpretation/Conversion:** The raw strokes are converted into TF parameters [G].
    *   *Neural Network Generation:* In some systems, strokes on volume slices are fed into an artificial neural network to generate high-dimensional TFs [S1, unofficial].
    *   *Feature Fusion:* Strokes on rendered images can fuse multiple features from distinct rendering results into a comprehensive visualization [S1, unofficial].
3.  **Refinement:** Users iteratively adjust the TF using specific interaction modes [S2, unofficial].
    *   *Sliding Window:* A rectangular pulse with controlled amplitude scans the domain. Opacity is constant within the interval and zero outside. Height and width are adjusted via cursors [S2, unofficial].
    *   *Mixing Table:* Vertical sliders associated with rectangular pulses. Ten equally sized sliders are placed side-by-side. Sliders are dragged to set height (opacity). Thumbnails act as toggle buttons for opacity (0 or 1) [S2, unofficial].
    *   *Ramp-like Function:* Piecewise linear interpolation between nodes. Nodes are moved vertically to adjust opacity [S2, unofficial].
    *   *Free-form Sketching:* Direct drawing on the TF graph updates the function values at the selected domain points [S2, unofficial].
4.  **Adaptation:** When switching tools, the prior TF is adapted to the new tool type. For example, switching from sketching to multiple sliders converts the function into a rectangular pulse train based on the average values in each slider's range [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Node:
    def __init__(self, scalar, opacity):
        self.scalar = scalar
        self.opacity = opacity

class TransferFunction:
    def __init__(self):
        self.nodes = []

    def update_from_stroke(self, stroke_points, tool_mode):
        if tool_mode == 'sketch':
            # Direct mapping: horizontal position maps to scalar, vertical to opacity [G]
            new_nodes = []
            for x, y in stroke_points:
                new_nodes.append(Node(x, y))
            self.nodes = sorted(new_nodes, key=lambda n: n.scalar)
            
        elif tool_mode == 'sliding_window':
            # stroke_points define window boundaries and height
            if len(stroke_points) >= 3:
                x_start, x_end, height = stroke_points[0][0], stroke_points[1][0], stroke_points[2][1]
                self._apply_rectangular_pulse(x_start, x_end, height)
                
        elif tool_mode == 'mixing_table':
            # Map stroke to nearest slider bin (0-9)
            if stroke_points:
                x = stroke_points[0][0]
                opacity = stroke_points[0][1]
                bin_index = int(x * 10) % 10
                self._set_slider_height(bin_index, opacity)
                
        elif tool_mode == 'spatial':
            # Map screen stroke to volume features [G]
            # Requires ray casting to map screen pixel to volume scalar
            volume_scalars = self._map_screen_to_volume(stroke_points)
            # Use NN or heuristic to generate TF from scalars [S1, unofficial]
            self._generate_tf_from_scalars(volume_scalars)
            
        return self.nodes

    def _apply_rectangular_pulse(self, x_start, x_end, height):
        # Opacity is constant within interval and zero outside [S2, unofficial]
        for node in self.nodes:
            if x_start <= node.scalar <= x_end:
                node.opacity = height
            else:
                node.opacity = 0.0
                
    def _set_slider_height(self, bin_index, opacity):
        # Ten equally sized sliders are placed side-by-side [S2, unofficial]
        if 0 <= bin_index < 10:
            for node in self.nodes:
                if abs(node.scalar - bin_index / 10.0) < 0.05:
                    node.opacity = opacity
                    return
            self.nodes.append(Node(bin_index / 10.0, opacity))
            
    def _map_screen_to_volume(self, points):
        # Conceptual mapping from screen coordinates to volume scalar values [G]
        return [0.5] * len(points)
        
    def _generate_tf_from_scalars(self, scalars):
        # Neural network or heuristic conversion [S1, unofficial]
        for s in scalars:
            self.nodes.append(Node(s, 0.5))
```

**Parameters:**
*   **Domain Resolution:** In Voxel TIPs, the mixing table uses 10 equally sized sliders [S2, unofficial].
*   **Thumbnail Interval:** Thumbnails represent bounded intervals of length 10% of the full parameter domain [S2, unofficial].

**Edge Cases:**
*   **Tool Switching:** Switching from sketching to sliders requires averaging values to create a rectangular pulse train, which may lose fine details [S2, unofficial].
*   **Precision:** Touch gestures on large screens (e.g., 55” TV) may have lower precision than mouse input, though users prefer them for intent reproduction [S2, unofficial].

## Subtopics

### [Interactive Volume Rendering with Multi-Dimensional TFs](6183-interactive-volume-rendering-with-multi-dimensional-tfs.md)
This method enables users to specify three-dimensional transfer functions (TFs) by interacting directly with a 2D histogram of scalar values versus gradient magnitudes. It solves the limitation of 1D TFs, which cannot accurately capture material boundaries where different materials share similar scalar values but differ in gradient characteristics. By allowing users to "paint" opacity regions in this 2D domain using geometric widgets, it provides a more intuitive specification process than manual parameter tuning.

### Sliding Window Scanning
A technique to scan the intensity and gradient magnitude domains to sketch initial concepts for intensity-based and gradient-based transfer function terms [S2, unofficial]. It applies a sliding window (rectangular pulse) to scan the domain, followed by configuring multiple sliders to assign nonzero opacity values to the region of interest [S2, unofficial].

### Sketch-based Feature Fusion
A method where, based on user sketches on the rendered images, multiple features in distinct rendering results are fused into a comprehensive visualization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time interaction is supported [S2, unofficial]. Stroke processing is $O(N)$ where $N$ is the number of stroke points [S2, unofficial]. Rendering updates depend on the ray marching algorithm [S2, unofficial].
*   **Memory Complexity:** Low [G]. TFs are stored as small sets of keypoints or LUTs [G].
*   **Performance:** Voxel TIPs was tested on a Samsung UE55F8000 TV with ZaagTech X Series 16 Touch Overlay [S2, unofficial]. When asked about the overall level of difficulty felt when designing the transfer function, users found it more difficult to sketch a function with a mouse, preferring touch gestures that more accurately reproduce the users intentions to sketch transfer functions [S2, unofficial].
*   **Strengths:** Intuitive, rapid initial definition, supports global and local editing, leverages direct manipulation metaphors (mixing table, sliding window) [S2, unofficial].
*   **Limitations:** Precision may be lower on large touch surfaces compared to mouse [S2, unofficial].
*   **When to use:** When users need to quickly explore TFs or have difficulty with abstract widget manipulation [S2, unofficial]. Suitable for collaborative or standing interactions (tabletops) [S2, unofficial].

## Background
Sketch-based TF editing emerged as an alternative to traditional WIMP (Window, Icon, Menu, Pointer) interfaces, which were often considered tedious for TF design [S2, unofficial]. Key developments include:
*   **Tzeng et al.:** Proposed systems allowing users to sketch volume slices to assign color and transparency, generating high-dimensional TFs using artificial neural networks [S1, unofficial].
*   **Ropinski et al.:** Proposed a sketch-based UI for 1D TF design where users draw strokes to define foreground/background layers, refined by adjusting color and opacity [S1, unofficial].
*   **Guo et al.:** Developed a system enabling direct manipulation of optical properties on the volume rendering view via user-drawn strokes, intelligently defining 1D TFs [S1, unofficial].
*   **Wu and Qu:** Proposed fusing multiple features from distinct rendering results based on user sketches on rendered images [S1, unofficial].
*   **Lopes et al. (Voxel TIPs):** Developed a sketch-based interface for a multi-touch tabletop, combining histograms, thumbnails, sliding windows, and mixing tables for intensity and gradient-based TF design [S2, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Explicit design of transfer functions for volume-rendered images, https://web.tecnico.ulisboa.pt/daniel.s.lopes/papers/Explicit%20design%20of%20transfer%20functions%20for%20volume-rendered%20images%20-%20Vis%20Comp%202017.pdf, fetched 2026-10-29 (unofficial)
