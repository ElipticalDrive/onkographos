# Transfer Function Control Point Management

Also called: Automatic TF Control Point Handling, TF Control Point Management

## Overview
This method manages transfer function control points with automatic sorting, duplicate handling, and lookup table (LUT) initialization based on the scalar range [S1, unofficial]. It sets itself apart by maintaining a synchronized LUT that maps scalar indices to RGBA colors for piecewise linear interpolation, ensuring consistent data integrity during control point modifications [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A scalar range $[s_{min}, s_{max}]$, a list of existing control points, and a new control point to insert [S1, unofficial].
- **Outputs**: An updated list of sorted control points and a rebuilt LUT array [S1, unofficial].

**Data Structures**
- **Control Point**: A structure containing `value` (scalar) and `color` (RGBA tuple) [S1, unofficial].
- **LUT**: An array of size $N = s_{max} - s_{min}$, where each entry stores the interpolated RGBA color for the corresponding scalar index [S1, unofficial].

**Steps**
1. **Initialization**:
   - Define the scalar range: $s_{range} = s_{max} - s_{min}$ [S1, unofficial].
   - Create boundary control points: one at $s_{min}$ with color $(0, 0, 0, 0)$ (black, transparent) and one at $s_{max}$ with color $(1, 1, 1, 1)$ (white, opaque) [S1, unofficial].
   - Initialize the LUT array with size $s_{range}$ [S1, unofficial].

2. **Control Point Insertion**:
   - **Precision Clamping**: Clamp the alpha component of the new control point's color to 2 decimal places using the formula:
     $$ \alpha_{clamped} = \frac{\lfloor \alpha \times 100 \rfloor}{100.0} $$
     [S1, unofficial].
   - **Validation**: If the new control point's value is outside $[s_{min}, s_{max}]$, reject it and return an error code (e.g., -1) [S1, unofficial].
   - **Search**: Find the insertion index by comparing the new value against existing control points. The list must remain sorted by value [S1, unofficial].
   - **Duplicate Handling**: If a control point with the same value already exists, replace it with the new one. Otherwise, insert the new point at the calculated index to maintain order [S1, unofficial].

3. **LUT Rebuild**:
   - After any modification (insertion, deletion, or update), rebuild the LUT [S1, unofficial].
   - Iterate through the sorted control points. For each segment between control point $i$ and $i+1$, interpolate the RGBA color for all scalar indices in that range [S1, unofficial].
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
        self.control_points.append(ControlPoint(self.s_min, 0, 0, 0, 0))
        self.control_points.append(ControlPoint(self.s_max, 1, 1, 1, 1))

    def add_control_point(self, value, r, g, b, a):
        if value < self.s_min or value > self.s_max:
            return -1
        a_clamped = math.floor(a * 100) / 100.0
        new_cp = ControlPoint(value, r, g, b, a_clamped)
        for i, cp in enumerate(self.control_points):
            if cp.value == value:
                self.control_points[i] = new_cp
                self._rebuild_lut()
                return 0
            if cp.value > value:
                self.control_points.insert(i, new_cp)
                self._rebuild_lut()
                return 0
        self.control_points.append(new_cp)
        self._rebuild_lut()
        return 0

    def _rebuild_lut(self):
        for i in range(self.s_range):
            scalar = self.s_min + i
            cp_left = self.control_points[0]
            cp_right = self.control_points[-1]
            for j in range(len(self.control_points) - 1):
                if self.control_points[j].value <= scalar <= self.control_points[j+1].value:
                    cp_left = self.control_points[j]
                    cp_right = self.control_points[j+1]
                    break
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
[TF Simplification](3947-tf-simplification.md): Reduces the number of line segments in a hand-drawn TF by approximating consecutive segments using a sliding error-tolerance window, thereby reducing computational overhead for ambient occlusion calculations [S1, unofficial]. It distinguishes itself by enforcing strict geometric constraints where simplified segments must pass through defined error windows around original vertices [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method originates from a raycasting rendering implementation, specifically handling visualization mapping for transfer functions [S1, unofficial]. It manages control points for piecewise linear interpolation lookup tables to ensure valid and sorted TFs for rendering [S1, unofficial].

## Sources
[S1] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
