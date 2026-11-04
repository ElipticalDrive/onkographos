# Transfer Function Control Point Management

Also called: Automatic TF Control Point Handling, TF Control Point Management

## Overview
This method manages the lifecycle of control points within a piecewise linear transfer function (TF), ensuring data integrity through automatic sorting, duplicate resolution, and boundary validation. It distinguishes itself from general curve fitting by enforcing strict geometric constraints on insertion and maintaining a synchronized Lookup Table (LUT) that maps scalar values to RGBA colors, specifically targeting the noise and inconsistency inherent in hand-drawn inputs [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A scalar range $[s_{min}, s_{max}]$, a list of existing control points, and a new control point to insert.
- **Outputs**: An updated list of sorted control points and a rebuilt LUT array.

**Data Structures**
- **Control Point**: A structure containing `value` (scalar), `color` (RGBA), and potentially metadata.
- **LUT**: An array of size $N = s_{max} - s_{min}$, where each entry stores the interpolated RGBA color for the corresponding scalar index.

**Steps**
1. **Initialization**:
   - Define the scalar range: $s_{range} = s_{max} - s_{min}$.
   - Create boundary control points: one at $s_{min}$ with color $(0, 0, 0, 0)$ (black, transparent) and one at $s_{max}$ with color $(1, 1, 1, 1)$ (white, opaque) [S1, unofficial].
   - Initialize the LUT array with size $s_{range}$ [S1, unofficial].

2. **Control Point Insertion**:
   - **Precision Clamping**: Clamp the alpha component of the new control point's color to 2 decimal places using the formula:
     $$ \alpha_{clamped} = \frac{\lfloor \alpha \times 100 \rfloor}{100.0} $$
     This reduces noise from hand-drawn inputs [S1, unofficial].
   - **Validation**: If the new control point's value is outside $[s_{min}, s_{max}]$, reject it and return an error code (e.g., -1) [S1, unofficial].
   - **Search**: Find the insertion index by comparing the new value against existing control points. The list must remain sorted by value.
   - **Duplicate Handling**: If a control point with the same value already exists, replace it with the new one. Otherwise, insert the new point at the calculated index to maintain order [S1, unofficial].

3. **LUT Rebuild**:
   - After any modification (insertion, deletion, or update), rebuild the LUT.
   - Iterate through the sorted control points. For each segment between control point $i$ and $i+1$, interpolate the RGBA color for all scalar indices in that range.
   - The interpolation is linear between the colors of the adjacent control points [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class ControlPoint:
    def __init__(self, value, r, g, b, a):
        self.value = value
        self.color = (r, g, b, a)

class TransferFunction:
    def __init__(self, s_min, s_max):
        self.s_min = s_min
        self.s_max = s_max
        self.s_range = int(s_max - s_min)
        self.control_points = []
        self.lut = [None] * self.s_range
        self._initialize_boundaries()
        self._rebuild_lut()

    def _initialize_boundaries(self):
        # Add min (black, transparent) and max (white, opaque)
        self.control_points.append(ControlPoint(self.s_min, 0, 0, 0, 0))
        self.control_points.append(ControlPoint(self.s_max, 1, 1, 1, 1))

    def add_control_point(self, value, r, g, b, a):
        # Validate range
        if value < self.s_min or value > self.s_max:
            return -1

        # Clamp alpha precision
        a_clamped = math.floor(a * 100) / 100.0

        # Find insertion index
        new_cp = ControlPoint(value, r, g, b, a_clamped)
        
        # Check for duplicates and find position
        for i, cp in enumerate(self.control_points):
            if cp.value == value:
                # Replace duplicate
                self.control_points[i] = new_cp
                self._rebuild_lut()
                return 0
            if cp.value > value:
                # Insert before this point
                self.control_points.insert(i, new_cp)
                self._rebuild_lut()
                return 0
        
        # Insert at end if not found
        self.control_points.append(new_cp)
        self._rebuild_lut()
        return 0

    def _rebuild_lut(self):
        # Initialize LUT with boundary colors
        # Note: Implementation details for interpolation are not fully specified in source
        # [G] Standard linear interpolation between control points is assumed
        for i in range(self.s_range):
            scalar = self.s_min + i
            # Find surrounding control points
            # [G] Binary search or linear scan to find cp_left and cp_right
            cp_left = self.control_points[0]
            cp_right = self.control_points[-1]
            
            for j in range(len(self.control_points) - 1):
                if self.control_points[j].value <= scalar <= self.control_points[j+1].value:
                    cp_left = self.control_points[j]
                    cp_right = self.control_points[j+1]
                    break
            
            # Linear interpolation
            if cp_left.value == cp_right.value:
                color = cp_left.color
            else:
                t = (scalar - cp_left.value) / (cp_right.value - cp_left.value)
                r = cp_left.color[0] + t * (cp_right.color[0] - cp_left.color[0])
                g = cp_left.color[1] + t * (cp_right.color[1] - cp_left.color[1])
                b = cp_left.color[2] + t * (cp_right.color[2] - cp_left.color[2])
                a = cp_left.color[3] + t * (cp_right.color[3] - cp_left.color[3])
                color = (r, g, b, a)
            
            self.lut[i] = color
```

**Parameters**
- **LUTsize**: Equal to $s_{range}$ ($s_{max} - s_{min}$) [S1, unofficial].
- **Alpha Precision**: Clamped to 2 decimal places [S1, unofficial].

**Edge Cases and Pitfalls**
- Control points outside $[s_{min}, s_{max}]$ are rejected [S1, unofficial].
- Precision errors in alpha values from hand-drawn inputs are mitigated by clamping [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity**: Insertion is $O(N)$ where $N$ is the number of control points (for search/insert). LUT rebuild is $O(M)$ where $M$ is the LUT size ($s_{range}$).
- **Memory Complexity**: $O(M)$ for the LUT storage.
- **Strengths**: Simplifies user interaction by handling sorting and duplicates automatically; reduces noise from manual input.
- **Limitations**: LUT rebuild on every change can be costly for large LUT sizes.
- **When to use**: When using piecewise linear TFs with manual control point placement.
- **When not to use**: When real-time updates with thousands of control points are required without LUT rebuilds.

## Background
This method is part of a raycasting rendering implementation, specifically handling the visualization mapping for transfer functions. It is designed to manage the control points for a piecewise linear interpolation lookup table, ensuring that the TF remains valid and sorted for efficient rendering [S1, unofficial].

## Sources
[S1] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
