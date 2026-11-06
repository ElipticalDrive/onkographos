# Transfer Function Core Data Structure

Also called: Transfer Function Management

## Overview
The Transfer Function Core Data Structure is an in-memory representation that manages the control points and parameters defining a volume rendering transfer function, mapping scalar voxel values to optical properties (color and opacity) [S1, S2, unofficial]. It solves the problem of maintaining a consistent, editable state for complex transfer functions composed of piecewise linear segments and Gaussian distributions, distinct from the runtime evaluation mechanisms used by [Real-time transfer function evaluation](1551-real-time-transfer-function-evaluation.md) or the [Fast Classification Algorithm](5289-fast-classification-algorithm.md). It is distinguished by its explicit separation of color controls, alpha controls, and Gaussian objects, which are aggregated into lookup tables for rendering [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User-defined control points (position, color/opacity), Gaussian parameters (mean, sigma, height), and a resolution parameter.
*   **Output:** Two primary data structures: `m_rgbaTable` (a lookup table of RGBA values) and `m_alphaArray` (a lookup table of opacity values), both indexed by normalized scalar value $v \in [0, 1]$.

**Data Structures:**
The core structure maintains the following fields [S1, unofficial]:

1.  **`m_rgbaTable`**: `std::vector<vec4f>` of size `resolution`. Stores pre-computed RGBA values.
    *   `vec4f` layout: `x` (r), `y` (g), `z` (b), `w` (a).
    *   Size: `sizeof(vec4f)` bytes per entry.
2.  **`m_alphaArray`**: `std::vector<float>` of size `resolution`. Stores pre-computed opacity values.
3.  **`m_colorControls`**: `std::vector<ColorControl>`. Stores color control points.
    *   `ColorControl` layout: `position` (float), `color` (vec3f).
    *   Constraint: `sizeof(ColorControl)` must equal `sizeof(vec4f)` [S1, unofficial].
    *   Entries are maintained in sorted order by `position` [S1, unofficial].
4.  **`m_alphaControls`**: `std::vector<AlphaControl>`. Stores alpha control points.
    *   `AlphaControl` layout: `pos` (vec2f).
    *   `vec2f` layout: `x` (position), `y` (opacity).
    *   Constraint: `sizeof(AlphaControl)` must equal `sizeof(vec2f)` [S1, unofficial].
5.  **`m_gaussianObjects`**: `std::vector<GaussianObject>`. Stores Gaussian distribution parameters for opacity.
    *   `GaussianObject` fields: `mean` (float), `sigma` (float), `heightFactor` (float), `alphaArray` (internal buffer, not part of core lookup).

**Numbered Steps:**

1.  **Initialization:**
    *   Allocate `m_rgbaTable` and `m_alphaArray` with size `resolution` (default 1024) [S1, unofficial].
    *   Initialize `window_left` to 0 and `window_width` to 1 [S2, unofficial].
    *   If no controls are provided, call `randomize()` or default constructors [S2, unofficial].
    *   **Pitfall:** `resolution` must be greater than 0 [S1, unofficial].

2.  **Adding Controls:**
    *   **Color Control:** Add a `ColorControl` to `m_colorControls`. The `position` is a normalized float in $[0, 1]$. The `color` is a `vec3f` (RGB).
    *   **Alpha Control:** Add an `AlphaControl` to `m_alphaControls`. The `pos` is a `vec2f` where `x` is position and `y` is opacity.
    *   **Gaussian Object:** Add a `GaussianObject` with `mean`, `sigma`, and `heightFactor`.

3.  **Updating the Lookup Tables:**
    *   **Update Color Map (`updateColorMap`):**
        *   Iterate through `m_rgbaTable` indices $i$ from 0 to `resolution - 1`.
        *   Calculate normalized position $v = i / (\text{resolution} - 1)$.
        *   Determine color $C(v)$ by interpolating between `ColorControl` points in `m_colorControls`. If only one control point exists, use its color. If multiple, perform linear interpolation between the surrounding control points [S1, unofficial].
        *   Set `m_rgbaTable[i] = vec4f(C(v).r, C(v).g, C(v).b, 1.0f)` (alpha is handled separately or set to 1.0 here, depending on implementation specifics; typically RGBA table stores color, alpha array stores opacity). *Note: The source implies `m_rgbaTable` stores RGBA, but `m_alphaArray` is separate. The update function likely sets RGB from color controls and leaves A or combines them. Based on `updateFromAlphaControls`, alpha is likely managed via `m_alphaArray`.*
    *   **Update Alpha Array (`updateFromAlphaControls`):**
        *   Iterate through `m_alphaArray` indices $i$.
        *   Calculate normalized position $v = i / (\text{resolution} - 1)$.
        *   Determine opacity $\alpha(v)$ by interpolating between `AlphaControl` points in `m_alphaControls`.
        *   Add contributions from `GaussianObject` in `m_gaussianObjects`. For each Gaussian $G_j$ with mean $\mu_j$, sigma $\sigma_j$, and height factor $h_j$:
            $$ \alpha_{gauss}(v) = h_j \cdot \exp\left(-\frac{(v - \mu_j)^2}{2\sigma_j^2}\right) $$
        *   Combine base alpha from controls and Gaussian contributions (typically additive or max, source implies additive combination for Gaussian bumps on top of base alpha).
        *   Set `m_alphaArray[i] = \alpha(v)`.

4.  **Removal and Modification:**
    *   Remove a control point by index from the respective vector (`m_colorControls`, `m_alphaControls`, or `m_gaussianObjects`).
    *   Call `updateColorMap()` and/or `updateFromAlphaControls()` to regenerate the lookup tables [S1, unofficial].

5.  **Clearing:**
    *   Clear all internal data structures (`m_rgbaTable`, `m_alphaArray`, `m_colorControls`, `m_alphaControls`, `m_gaussianObjects`) [S1, unofficial].
    *   Reset alpha table to 0.0f [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class TransferFunctionCore:
    def __init__(self, resolution=1024):
        if resolution <= 0:
            raise ValueError("Resolution must be greater than 0")
        self.resolution = resolution
        self.m_rgbaTable = [vec4f(0, 0, 0, 1) for _ in range(resolution)]
        self.m_alphaArray = [0.0] * resolution
        self.m_colorControls = []  # List of ColorControl
        self.m_alphaControls = []  # List of AlphaControl
        self.m_gaussianObjects = [] # List of GaussianObject
        self.window_left = 0.0
        self.window_width = 1.0
        self.randomize() # Initialize with random defaults if needed

    def add_color_control(self, position, r, g, b):
        control = ColorControl(position, vec3f(r, g, b))
        self.m_colorControls.append(control)
        self.m_colorControls.sort(key=lambda c: c.position)
        self.update_color_map()

    def add_alpha_control(self, position, opacity):
        control = AlphaControl(vec2f(position, opacity))
        self.m_alphaControls.append(control)
        self.update_alpha_array()

    def add_gaussian_object(self, mean, sigma, height_factor):
        gauss = GaussianObject(mean, sigma, height_factor)
        self.m_gaussianObjects.append(gauss)
        self.update_alpha_array()

    def update_color_map(self):
        for i in range(self.resolution):
            v = i / (self.resolution - 1)
            color = self.interpolate_color(v)
            self.m_rgbaTable[i] = vec4f(color.x, color.y, color.z, 1.0)

    def update_alpha_array(self):
        for i in range(self.resolution):
            v = i / (self.resolution - 1)
            alpha = self.interpolate_alpha(v)
            for gauss in self.m_gaussianObjects:
                alpha += gauss.height_factor * math.exp(-((v - gauss.mean)**2) / (2 * gauss.sigma**2))
            self.m_alphaArray[i] = alpha

    def interpolate_color(self, v):
        if not self.m_colorControls:
            return vec3f(0, 0, 0)
        if len(self.m_colorControls) == 1:
            return self.m_colorControls[0].color
        # Find surrounding controls
        for j in range(len(self.m_colorControls) - 1):
            c1 = self.m_colorControls[j]
            c2 = self.m_colorControls[j+1]
            if c1.position <= v <= c2.position:
                t = (v - c1.position) / (c2.position - c1.position) if c2.position != c1.position else 0
                return c1.color + t * (c2.color - c1.color)
        # Fallback to last control
        return self.m_colorControls[-1].color

    def interpolate_alpha(self, v):
        if not self.m_alphaControls:
            return 0.0
        if len(self.m_alphaControls) == 1:
            return self.m_alphaControls[0].pos.y
        # Find surrounding controls
        for j in range(len(self.m_alphaControls) - 1):
            c1 = self.m_alphaControls[j]
            c2 = self.m_alphaControls[j+1]
            if c1.pos.x <= v <= c2.pos.x:
                t = (v - c1.pos.x) / (c2.pos.x - c1.pos.x) if c2.pos.x != c1.pos.x else 0
                return c1.pos.y + t * (c2.pos.y - c1.pos.y)
        # Fallback to last control
        return self.m_alphaControls[-1].pos.y

    def clear(self):
        self.m_rgbaTable = []
        self.m_alphaArray = []
        self.m_colorControls = []
        self.m_alphaControls = []
        self.m_gaussianObjects = []

    def randomize(self):
        # Implementation details not fully specified, assumes default initialization
        pass

class ColorControl:
    def __init__(self, position, color):
        self.position = position
        self.color = color # vec3f

class AlphaControl:
    def __init__(self, pos):
        self.pos = pos # vec2f

class GaussianObject:
    def __init__(self, mean, sigma, height_factor):
        self.mean = mean
        self.sigma = sigma
        self.height_factor = height_factor
        self.alphaArray = [] # Internal buffer, not part of core lookup

class vec2f:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class vec3f:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class vec4f:
    def __init__(self, x, y, z, w):
        self.x = x
        self.y = y
        self.z = z
        self.w = w
```

## Subtopics

### [Transfer Function Serialization](4343-transfer-function-serialization.md)
Converts the in-memory structure to JSON for persistence. Chosen when saving/loading configurations.

## Operating characteristics
*   **Time Complexity:** Update operations are $O(R)$ where $R$ is the resolution, as the entire lookup table must be regenerated when controls change. Interpolation during rendering is $O(1)$ per sample.
*   **Memory Complexity:** $O(R)$ for the lookup tables. Control points are $O(N)$ where $N$ is the number of controls.
*   **Strengths:** Fast runtime evaluation via texture lookup; flexible composition of linear and Gaussian segments.
*   **Limitations:** Regenerating the LUT on every control change can be costly if done frequently; resolution limits the precision of the mapping.
*   **When to use:** When real-time evaluation is required and the transfer function is relatively static between user interactions.

## Background
The structure originates from implementations in volume rendering frameworks such as OSPRay and related research projects [S1, S2, unofficial]. It is designed to bridge the gap between user interaction (adding control points) and GPU rendering (sampling LUTs).

## Sources
[S1] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S2] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
