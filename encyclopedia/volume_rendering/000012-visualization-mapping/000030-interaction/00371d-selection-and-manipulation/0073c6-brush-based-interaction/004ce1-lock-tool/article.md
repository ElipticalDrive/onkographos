# Lock Tool
Also called: Lock Brush
## Overview
The Lock Tool is an interaction mechanism that specifies data points immune to modification by the brush or dig tools, allowing users to protect specific data subsets during exploration [S1].
## How it works
**Inputs and Outputs:**
- Input: User mouse events (click, drag, double-click, scroll) and current lock state [S1].
- Output: Updated lock set and rendering visibility mask [S1].

**Data Structures:**
- Lock Set $L$: A set of indices or identifiers corresponding to data points (voxels or samples) currently marked as locked [S1].

**Steps:**
1. **Initialize Lock Set:** Start with an empty set or inherit from previous state [S1].
2. **Add to Lock Set:** When the user presses `Shift` + `Left Mouse Button`, identify points within the brush radius $r$ and add them to $L$ [S1]. Formula: $L_{new} = L_{old} \cup \{p \mid \|p - c\| \le r\}$, where $c$ is the brush center and $r$ is the brush radius [S1].
3. **Remove from Lock Set:** When the user presses `Control` + `Left Mouse Button`, identify points within the brush radius $r$ and remove them from $L$ [S1]. Formula: $L_{new} = L_{old} \setminus \{p \mid \|p - c\| \le r\}$ [S1].
4. **Adjust Brush Size:** Use the mouse wheel to adjust the brush radius $r$ [S1].
5. **Invert Lock Set:** Double-clicking in a view inverts the lock set, making all unlocked points locked and all locked points unlocked [S1]. Formula: $L_{new} = P_{all} \setminus L_{old}$, where $P_{all}$ is the set of all data points in the view [S1].
6. **Rendering:** Render points $p \in L$ as visible. Render points $p \notin L$ as not drawn (invisible) [S1].

**Parameters:**
- Brush Radius $r$: Adjustable via mouse wheel [S1].
- Modifier Keys: `Shift` for Add, `Control` for Remove [S1].

**Edge Cases and Pitfalls:**
- **Degenerate Input:** If $L$ contains all points, inversion results in an empty set. If $L$ is empty, inversion results in all points [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
SHIFT = 0x01
CONTROL = 0x02

def draw(point):
    # Standard graphics API call to render the point
    pass

class LockTool:
    def __init__(self, all_points):
        self.all_points = all_points
        self.locked_set = set()
        self.brush_radius = 5.0

    def on_mouse_wheel(self, delta):
        self.brush_radius += delta * 0.5
        if self.brush_radius < 1.0:
            self.brush_radius = 1.0

    def on_mouse_down(self, x, y, modifiers):
        is_shift = bool(modifiers & SHIFT)
        is_ctrl = bool(modifiers & CONTROL)
        points_in_brush = self.get_points_in_radius(x, y, self.brush_radius)
        if is_shift:
            self.locked_set.update(points_in_brush)
        elif is_ctrl:
            self.locked_set -= points_in_brush

    def on_double_click(self, x, y):
        self.locked_set = set(self.all_points) - self.locked_set

    def get_points_in_radius(self, x, y, radius):
        # Project (x,y) to 3D, find points within distance radius
        return {p for p in self.all_points if self.distance(p, (x,y)) <= radius}

    def render(self):
        for point in self.all_points:
            if point in self.locked_set:
                draw(point)
```
## Subtopics

### Filtering Brush
A tool to remove uninteresting data points or reduce clutter, combined with locking to selectively affect only locked or unlocked points [S1]. It uses add and remove modes similar to the Lock Tool but with inverted logic regarding which set is affected [S1]. Remove mode (`Ctrl` + `Left Mouse Button`) removes points, but only affects **locked** points [S1]. Add mode (`Shift` + `Left Mouse Button`) adds back previously removed points, but only affects **unlocked** points [S1]. Brush size is adjusted via mouse wheel [S1]. This allows users to first lock high-density regions and then filter out low-density noise, or vice versa, to reveal structures like skull anatomy in head scans [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Simple mental model, effective for isolating specific features by hiding clutter [S1].
- **Limitations:** Binary visibility (on/off) may obscure context. Inversion can be disruptive if not intended [S1].
- **When to Use:** When exploring volumetric datasets with significant noise or uninteresting background data, and the user needs to focus on specific subsets by hiding the rest [S1].
- **When Not to Use:** Not given in the sources.
## Background
Developed to allow users to "peek" inside volumetric data, such as human head scans, by locking interesting structures (like the skull) and removing uninteresting noise (like soft tissue) [S1].
## Sources
[S1] Color Tunneling: Interactive Exploration and Selection in Volumetric Datasets, https://doi.org/10.1109/pacificvis.2014.61, fetched 2026-10-26
