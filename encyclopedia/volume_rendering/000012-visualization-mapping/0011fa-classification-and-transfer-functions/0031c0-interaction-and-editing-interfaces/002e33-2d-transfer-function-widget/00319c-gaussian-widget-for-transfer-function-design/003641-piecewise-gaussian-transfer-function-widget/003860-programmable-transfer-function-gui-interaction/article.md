# Programmable Transfer Function GUI Interaction

Also called: Dynamic TF Parameter Modification, IMGUI TF Interaction, ImGui Transfer Function Widget

## Overview
An interactive UI widget for editing volume rendering transfer functions using draggable control points organized per color channel (R, G, B, A) [S1, unofficial]. It allows users to dynamically modify variables in Programmable Transfer Functions (PTFs), such as isovalue, shading color, volume step size, silhouette terms, and specular highlight coefficients, without recomputing and storing new transfer function textures [S2]. Unlike traditional methods that require recomputing and storing new transfer function textures, this approach enables real-time tuning of rendering parameters via a simple widget interface [S2].

## How it works
**Inputs and Outputs:**
- **Inputs:** Mouse events (hover, click, drag, release) within a canvas area; current state of handles.
- **Outputs:** Modified handle positions in normalized canvas coordinates; a boolean flag indicating if the transfer function changed; a `std::vector<float>` containing RGBA data suitable for `glTextureSubImage1D` [S1, unofficial].

**Data Structures:**
- `groups_`: An array of 4 `std::vector<ImVec2>`, one for each channel (R, G, B, A). Each `ImVec2` stores normalized canvas coordinates $(x, y)$ where $x \in [0, 1]$ represents the domain and $y \in [0, 1]$ represents the value [S1, unofficial].
- `domain_` (int): Number of samples in the transfer function domain [S1, unofficial].
- `width_`, `height_` (float): Canvas dimensions in pixels [S1, unofficial].
- `scaling_` (float): UI scaling factor for handle radius and margins. Default: 1.6f [S1, unofficial].
- `handleRadius_` (float): Base radius for drawing handles. Calculated as $4.0f \times scaling_$ [S1, unofficial].
- `handleBorder_` (float): Border width for handles. Calculated as $1.0f \times scaling_$ [S1, unofficial].
- `handleCountMax_` (int): Maximum number of handles allowed per channel group. Default: 20 [S1, unofficial].
- `margins_` (ImVec2): Padding around the canvas area. Calculated as $(2 \times handleRadius_, 2 \times handleRadius_)$ [S1, unofficial].
- `groupActive_` (int): Index of the currently active channel group (0: R, 1: G, 2: B, 3: A). Default: A (3) [S1, unofficial].
- `handleActive_` (int): Index of the currently captured handle within the active group [S1, unofficial].
- `isHandleCaptured_` (bool): True if a handle is currently being dragged [S1, unofficial].
- `isFocusMode_` (bool): If true, prevents switching active groups by clicking other handles [S1, unofficial].

**Numbered Steps:**

1. **Initialization:**
   - Set `domain_`, `width_`, `height_`.
   - Calculate `handleRadius_ = 4.0f * scaling_`, `handleBorder_ = 1.0f * scaling_`, `margins_ = (2 * handleRadius_, 2 * handleRadius_)`.
   - Initialize default handles for each channel in normalized canvas coordinates [S1, unofficial]:
     - **R:** $(0.625, 0.38), (0.75, 0.5), (0.875, 0.75), (1.0, 1.0)$
     - **G:** $(0.0, 0.0), (0.125, 0.85), (0.25, 1.0), (0.5, 0.88), (0.75, 0.0), (1.0, 0.0)$
     - **B:** $(0.0, 1.0), (0.125, 0.5), (0.24, 0.25), (0.38, 0.125)$
     - **A:** $(0.0, 0.0625), (0.25, 0.0625), (0.5, 0.0625), (0.75, 0.0625), (1.0, 0.0625)$

2. **Rendering (Draw Order):**
   - **Non-active channel lines:** Iterate through all groups except the active one. For each non-empty group, draw lines between consecutive handles using `drawList->AddLine(screenCoord(group[p]), screenCoord(group[p+1]), lineColor, 1.2f)`. In focus mode, lines have 0.6 alpha [S1, unofficial].
   - **Non-active channel handles:** For each handle in non-active groups, draw an outer white circle `drawList->AddCircleFilled(pos, handleRadius_, lineColor)` and an inner colored circle `drawList->AddCircleFilled(pos, handleRadius_ - handleBorder_, handleColor)`. In focus mode, both have 0.5 alpha [S1, unofficial].
   - **Active channel:**
     - Scale active handle radius by $1.35f$ (`activeRadius = 1.35f * handleRadius_`).
     - Draw lines with width 1.5f.
     - Draw outer white circle with `activeRadius`.
     - Draw inner colored circle with 0.6 alpha [S1, unofficial].

3. **Event Detection:**
   - Calculate `radiusWithinReach = 4 * handleRadius_ * handleRadius_ / (width_ * width_ + height_ * height_)` [S1, unofficial].
   - Convert mouse position to canvas coordinates.
   - Find the closest handle in the active group by squared Euclidean distance in canvas coordinates.
   - If the closest handle distance is less than `radiusWithinReach`:
     - Show tooltip `(%.0f, %.4f)` with domain-scaled X (`domain_ * x`) and raw Y [S1, unofficial].
     - On left-click, capture the handle (`isHandleCaptured_ = true`, `handleActive_ = index`) [S1, unofficial].
   - If no active handle is reachable:
     - Search other groups for the closest handle.
     - If a handle in another group is reachable:
       - Show tooltip for that handle [S1, unofficial].
       - If left-clicked and `isFocusMode_` is false, switch the active group to that channel (`setActiveGroup(groupIndex)`) and do not capture a handle [S1, unofficial].
       - If left-clicked and `isFocusMode_` is true, do nothing [S1, unofficial].

4. **Handle Manipulation:**
   - On mouse release, set `isHandleCaptured_ = false` [S1, unofficial].
   - If a handle is captured:
     - **Right-click:** Erase the handle if `active.size() > 1`. Set `changed = true` [S1, unofficial].
     - **Left-drag:**
       - Clamp X coordinate between neighbors' X coordinates: `xMin = active[handleActive_-1].x`, `xMax = active[handleActive_+1].x`. If at boundary, clamp to 0 or 1 respectively [S1, unofficial].
       - Clamp Y coordinate to $[0.0, 1.0]$ [S1, unofficial].
       - Update handle position [S1, unofficial].
   - If not captured and right-clicked inside canvas bounds and `active.size() < handleCountMax_`:
     - Append new handle at mouse position.
     - Sort all handles in the active group by X coordinate [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def handle_events(mouse_pos, canvas_width, canvas_height, groups, group_active, handle_active, is_handle_captured, is_focus_mode, domain, handle_count_max, handle_radius, width, height):
    # Convert mouse pos to canvas coordinates (normalized 0-1)
    canvas_mouse = ImVec2(mouse_pos.x / canvas_width, mouse_pos.y / canvas_height)
    
    # Calculate reach radius squared
    radius_within_reach = 4.0 * handle_radius * handle_radius / (width * width + height * height)
    
    active_group = groups[group_active]
    changed = False
    
    if is_handle_captured:
        # Handle dragging and deletion
        if is_right_click():
            if len(active_group) > 1:
                del active_group[handle_active]
                changed = True
                is_handle_captured = False
        elif is_left_dragging():
            # Clamp X
            if handle_active > 0:
                x_min = active_group[handle_active - 1].x
            else:
                x_min = 0.0
            if handle_active < len(active_group) - 1:
                x_max = active_group[handle_active + 1].x
            else:
                x_max = 1.0
            
            active_group[handle_active].x = clamp(canvas_mouse.x, x_min, x_max)
            active_group[handle_active].y = clamp(canvas_mouse.y, 0.0, 1.0)
            
            # Re-sort if necessary (though clamping preserves order)
            active_group.sort(key=lambda p: p.x)
            
        if is_left_release():
            is_handle_captured = False
            
    else:
        # Find closest handle in active group
        min_dist_sqr = float('inf')
        handle_near = -1
        
        for i, p in enumerate(active_group):
            dist_sqr = (canvas_mouse.x - p.x)**2 + (canvas_mouse.y - p.y)**2
            if dist_sqr < min_dist_sqr:
                min_dist_sqr = dist_sqr
                handle_near = i
                
        if min_dist_sqr < radius_within_reach:
            show_tooltip(f"({domain * active_group[handle_near].x:.0f}, {active_group[handle_near].y:.4f})")
            if is_left_click():
                handle_active = handle_near
                is_handle_captured = True
        else:
            # Search other groups
            group_near = -1
            handle_near_other = -1
            min_dist_sqr_other = float('inf')
            
            for g_idx, group in enumerate(groups):
                if g_idx == group_active:
                    continue
                for h_idx, p in enumerate(group):
                    dist_sqr = (canvas_mouse.x - p.x)**2 + (canvas_mouse.y - p.y)**2
                    if dist_sqr < min_dist_sqr_other:
                        min_dist_sqr_other = dist_sqr
                        group_near = g_idx
                        handle_near_other = h_idx
                        
            if min_dist_sqr_other < radius_within_reach:
                show_tooltip(f"({domain * groups[group_near][handle_near_other].x:.0f}, {groups[group_near][handle_near_other].y:.4f})")
                if is_left_click():
                    if not is_focus_mode:
                        group_active = group_near
                        is_handle_captured = False
                    # If focus mode, do nothing
                elif is_right_click():
                    if len(groups[group_near]) < handle_count_max:
                        groups[group_near].append(canvas_mouse)
                        groups[group_near].sort(key=lambda p: p.x)
                        changed = True
```

**Parameters:**
- `isovalue`: The value used to render the surface, modifiable via the GUI [S2].
- `volume step size`: The step size for ray marching, modifiable via the GUI [S2].
- `silhouette term`: A parameter of the Programmable Transfer Function, modifiable via the GUI [S2].
- `silhouette coefficients`: Coefficients for the silhouette shading, modifiable via the GUI [S2].
- `coefficient of specular highlight augmentation`: The $\mu_{spec}$ parameter, modifiable via the GUI [S2].
- `intersection shading toggle`: Toggle intersection shading on or off, modifiable via the GUI [S2].

**Edge Cases and Pitfalls:**
- Dragging a handle horizontally must preserve ordering: X is clamped to the range defined by adjacent handles to prevent crossing [S1, unofficial].
- Y coordinate is strictly clamped to $[0.0, 1.0]$ to prevent out-of-bounds transfer function values [S1, unofficial].
- If `isFocusMode_` is true, clicking on handles in non-active groups does not switch the active group [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Event handling involves iterating through handles to find the closest one. With $N$ handles per group and 4 groups, the complexity is $O(N)$ per frame for event detection. Rendering is also $O(N)$ for drawing lines and circles.
- **Memory Complexity:** $O(N)$ to store handle positions, where $N$ is the maximum number of handles per channel (default 20) [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Allows real-time modification of PTF parameters without recomputing textures, providing immediate visual feedback [S2]. The widget is lightweight and integrates with ImGui [S1, unofficial].
- **Limitations:** Limited to 2D interaction; complex 3D transfer functions may require more sophisticated interaction models. The default handle count is limited to 20 per channel [S1, unofficial].
- **When to use:** When using Programmable Transfer Functions and needing real-time, interactive tuning of color, opacity, and shading parameters [S2].
- **When not to use:** When high-performance, non-interactive batch processing is required, or when 3D spatial transfer functions are needed.

## Background
The Programmable Transfer Function (PTF) concept was introduced to allow flexible, real-time modification of volume rendering parameters [S2]. The specific GUI interaction widget using ImGui was implemented as part of a system for visualizing WiFi signals, demonstrating the versatility of PTFs for dynamic parameter tuning [S2]. The widget allows users to modify isovalue, shading color, volume step size, silhouette terms, and specular highlight coefficients without the overhead of texture recomputation [S2].

## Sources
[S1] kogiokka/imgui-transfer-function, https://github.com/kogiokka/imgui-transfer-function, fetched 2026-10-27 (unofficial)
[S2] Visualization of WiFi Signals Using Programmable Transfer Functions, https://doi.org/10.3390/info13050224, fetched 2026-10-27
