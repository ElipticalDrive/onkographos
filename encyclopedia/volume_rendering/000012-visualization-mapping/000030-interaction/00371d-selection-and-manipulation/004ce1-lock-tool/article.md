# Lock Tool
Also called: Lock Brush
## Overview
The Lock Tool is an interaction mechanism that specifies data points immune to modification by subsequent brush or dig tools, allowing users to protect specific data subsets during volumetric exploration [S1]. It distinguishes itself from siblings like [Brushing and Linking with Hilbert Codes](10912-brushing-and-linking-with-hilbert-codes.md) by operating on a binary visibility state (locked/unlocked) rather than continuous selection weights, and from [3D Widget Interaction Model](11643-3d-widget-interaction-model.md) by focusing on data subset protection rather than geometric transformation. It solves the problem of isolating regions of interest from accidental modification or visual clutter during interactive volume rendering [S1].
## How it works
**Inputs and Outputs:**
- **Input:** User gestures (mouse movement, click, double-click, scroll) in a 2D view projection of the volume.
- **Output:** A binary state map for data points (locked or unlocked) and a rendering update where locked points are visible and unlocked points are invisible.

**Data Structures:**
- **Lock State Map:** A boolean array or bitmask aligned with the data points (voxels or particles). Let $L$ be the set of locked indices.
  - $L_i = \text{true}$ if point $i$ is locked.
  - $L_i = \text{false}$ if point $i$ is unlocked.
  - Initialization: All points are unlocked ($L_i = \text{false}$) by default.

**Parameters:**
- **Brush Radius ($r$):** The spatial extent of the selection tool.
  - Typical value: Adjustable via mouse wheel. Not given in the sources.
- **Key Modifiers:**
  - `Shift`: Invokes Add mode.
  - `Control`: Invokes Remove mode.
  - `None` (Double-click): Invokes Invert mode.

**Steps:**
1. **Initialize:** Set all data points to unlocked state.
2. **Select Mode:**
   - If `Shift` + Left Mouse Button is pressed: Enter **Add Mode**.
   - If `Control` + Left Mouse Button is pressed: Enter **Remove Mode**.
   - If Left Mouse Button is double-clicked: Enter **Invert Mode**.
3. **Adjust Brush Size:**
   - Scroll mouse wheel to adjust brush radius $r$.
4. **Execute Operation:**
   - **Add Mode:** For all data points $p$ within the brush radius $r$ in the current view:
     - Set $L_p = \text{true}$.
   - **Remove Mode:** For all data points $p$ within the brush radius $r$ in the current view:
     - Set $L_p = \text{false}$.
   - **Invert Mode:** For all data points $p$ in the current view:
     - Set $L_p = \neg L_p$.
5. **Render:**
   - For each data point $p$:
     - If $L_p$ is true, render $p$ (visible).
     - If $L_p$ is false, do not render $p$ (invisible).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
lock_state = {p: False for p in data_points} # All unlocked initially
brush_radius = 10.0 # Initial radius

def handle_mouse_move(event):
    """Adjust brush size."""
    global brush_radius
    brush_radius += event.scroll_delta * scale_factor

def handle_mouse_click(event):
    """Handle single click for add/remove."""
    if event.button == LEFT:
        if event.modifiers & SHIFT:
            mode = ADD
        elif event.modifiers & CONTROL:
            mode = REMOVE
        else:
            return # No action for simple click without modifier
        
        # Get points within brush radius in current view
        points_in_brush = get_points_in_radius(event.position, brush_radius)
        
        for p in points_in_brush:
            if mode == ADD:
                lock_state[p] = True
            elif mode == REMOVE:
                lock_state[p] = False

def handle_mouse_double_click(event):
    """Handle double click for invert."""
    if event.button == LEFT:
        points_in_view = get_all_points_in_view(event.view)
        for p in points_in_view:
            lock_state[p] = not lock_state[p]

def render_volume():
    """Render based on lock state."""
    for p in data_points:
        if lock_state[p]:
            draw(p)
        else:
            skip(p)
```

**Edge Cases and Pitfalls:**
- **Precision:** The definition of "within the brush radius" depends on the projection space (screen space vs. world space). The sources imply screen-space interaction in a 2D view [S1].
- **Boundaries:** The lock state is persistent across view changes unless explicitly inverted or modified.
- **Degenerate Input:** Double-clicking in an empty view has no effect.

## Subtopics

### Filtering Brush
A variant of the Lock Tool that removes uninteresting data points or reduces clutter, combined with locking to selectively affect only locked or unlocked points. It differs from the standard Lock Tool by having conditional application rules based on the current lock state.
- **Definition:** A tool to remove uninteresting data points or reduce clutter, combined with locking to selectively affect only locked or unlocked points.
- **Steps:**
  1. Use the filtering brush with add and remove modes.
  2. Remove mode (Ctrl + Left Mouse Button) removes points.
  3. Add mode (Shift + Left Mouse Button) adds back previously removed points.
  4. Adjust the filtering brush size using the mouse wheel.
  5. In remove mode, only locked points are affected by the filtering brush.
  6. In add mode, only unlocked points are affected.
- **When to choose:** Use when the goal is to hide clutter (e.g., noise) while preserving the ability to restore it, specifically targeting regions that have already been locked or unlocked.
- **Link:** [Filtering Brush](10913-filtering-brush.md) (Hypothetical sibling, not provided in tree, so link to parent if needed, but instructions say "link to it" if it has an article. Since no article is provided for Filtering Brush, we omit the link or link to parent. The prompt says "Links to other articles use the file names given below". No file name is given for Filtering Brush. So we leave it without a link or link to the main article if appropriate. However, the instruction says "one entry per child... and its link". If no link exists, we cannot provide one. We will omit the link.)

## Operating characteristics
- **Time Complexity:** $O(N)$ per frame for rendering, where $N$ is the number of data points, as each point must be checked against its lock state. Selection operations are $O(K)$, where $K$ is the number of points within the brush radius.
- **Memory Complexity:** $O(N)$ to store the lock state for each data point.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades linearly with the number of data points. The lock state is independent of resolution, relying on data point count.
- **Strengths:** Simple binary state is easy to understand and implement. Effective for isolating regions of interest.
- **Limitations:** No gradual transition between locked and unlocked states. All unlocked points are completely invisible, which may hide context.
- **When to use:** When the user needs to permanently hide certain data subsets during an exploration session.
- **When not to use:** When gradual opacity or partial selection is required.

## Background
The Lock Tool was introduced in the context of interactive exploration and selection in volumetric datasets, specifically in the "Color Tunneling" method [S1]. It was developed to allow users to "peek" inside complex volumetric data, such as human head scans, by locking interesting structures (like the skull) and unlocking or removing uninteresting noise (like soft tissue) [S1]. The method was presented in a paper published in 2014 [S1].

## Sources
[S1] Color Tunneling: Interactive Exploration and Selection in Volumetric Datasets, https://doi.org/10.1109/pacificvis.2014.61, fetched 2026-10-26
