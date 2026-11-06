# Cubic Spline Transfer Function with Indexed Caching

Also called: TF Management with Cubic Splines and Indexed Array, Transfer Function Management and Evaluation

## Overview
This method manages 1D transfer functions for volume rendering by mapping scalar density values (0–255) to RGBA colors using cubic spline interpolation of control points, while caching the results in a 256-entry indexed array for fast integer lookup [S1, unofficial]. It builds on [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md) by adding an explicit indexed cache and lazy texture synchronization, distinguishing itself from standard spline implementations by decoupling the smooth float-based interpolation logic from the discrete integer-based rendering lookup [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Scalar density value (float $t \in [0, 1]$ or integer $isoValue \in [0, 255]$), control points for color and alpha.
- **Outputs:** RGBA color vector.

**Data Structures:**
The implementation maintains the following structures:
- `colorPoints`: `std::vector<TransferFunctionColorPoint>`. Each point contains an `isoValue` (int) and `color` (`glm::vec3`).
- `alphaPoints`: `std::vector<TransferFunctionAlphaPoint>`. Each point contains an `isoValue` (int) and `alpha` (float).
- `colorSpline`: `std::vector<CubicSpline>`. Interpolates color values between color points.
- `alphaSpline`: `std::vector<CubicSpline>`. Interpolates alpha values between alpha points.
- `indexedTransferFunction`: `std::array<vec4, 256>`. Caches the final RGBA result for each integer iso value 0–255.
- `colorMappingTexture`: `gl::Texture1dRef`. A 1D OpenGL texture (GL_RGBA, GL_FLOAT, size 256) used for GPU access.
- `threshold`: `vec2`. Clamps the effective range of iso values.

**Initialization:**
1. Set threshold to $(0, 255)$.
2. Add two default color points: $(vec3(1), 0)$ and $(vec3(1), 255)$.
3. Add two default alpha points: $(1.0, 0)$ and $(1.0, 255)$.
4. Call `updateFunction()` to initialize splines and cache.

**Adding a Color Point:**
1. Check validity: `isoValue > 0`, `isoValue < colorPoints.back().getIsoValue()`, and `colorPoints.size() < 255`.
2. Create `TransferFunctionColorPoint` with color and iso value.
3. Insert into `colorPoints` using `upper_bound` to maintain sorted order by `isoValue`.
4. Call `updateFunction()`.

**Adding an Alpha Point:**
1. Check validity: `isoValue > 0`, `isoValue < alphaPoints.back().getIsoValue()`, and `alphaPoints.size() < 255`.
2. Create `TransferFunctionAlphaPoint` with alpha and iso value.
3. Insert into `alphaPoints` using `upper_bound` to maintain sorted order by `isoValue`.
4. Call `updateFunction()`.

**Removing a Color Point:**
1. Check validity: `index > 0` and `index < colorPoints.size()`.
2. Erase element at `index` from `colorPoints`.
3. Call `updateFunction()`.

**Removing an Alpha Point:**
1. Check validity: `index > 0` and `index < alphaPoints.size()`.
2. Erase element at `index` from `alphaPoints`.
3. Call `updateFunction()`.

**Setting Alpha Value:**
1. Check validity: `index >= 0` and `index < alphaPoints.size()`.
2. Set alpha to `clamp(alpha, 0.0f, 1.0f)`.
3. Call `updateFunction()`.

**Setting Threshold:**
1. Clamp `minIso` to $[0, maxIso - 1]$.
2. Clamp `maxIso` to $[minIso + 1, 255]$.
3. Store in `threshold.x` and `threshold.y`.

**Setting Color:**
1. Check validity: `index >= 0` and `index < colorPoints.size()`.
2. Set color to `clamp(color, vec3(0), vec3(1))`.
3. Call `updateFunction()`.

**Setting Alpha Point Iso Value:**
1. Check validity: `index > 0`, `index < alphaPoints.size()`, `isoValue > 0`, and `isoValue < 255`.
2. Set iso value.
3. If `alphaPoints` is not sorted, sort it.
4. Call `updateFunction()`.

**Setting Color Point Iso Value:**
1. Check validity: `index > 0`, `index < colorPoints.size()`, `isoValue > 0`, and `isoValue < 255`.
2. Set iso value.
3. If `colorPoints` is not sorted, sort it.
4. Call `updateFunction()`.

**Updating Texture:**
1. If `colorMappingTexture` is null, create a 1D texture with format GL_RGBA, internal format GL_RGBA, data type GL_FLOAT, min/mag filter GL_LINEAR, wrap GL_CLAMP_TO_EDGE, size 256, initialized with `indexedTransferFunction.data()`.
2. If `updateColorTexture` is true, update texture data with `indexedTransferFunction.data()`, GL_RGBA, GL_FLOAT, size 256.
3. Set `updateColorTexture = true` in `updateFunction()` to trigger this update.

**Evaluating Color by Float $t$:**
1. Convert $t$ to integer: $isoValue = \lfloor 255.0 \times t \rfloor$.
2. Iterate through `alphaPoints` to find the interval $[currentIso, nextIso]$ containing $isoValue$.
3. Calculate interpolation factor: $evalAt = \frac{isoValue - currentIso}{nextIso - currentIso}$.
4. Get alpha from `alphaSpline[i].getPointOnSpline(evalAt).r`.
5. Repeat for color using `colorPoints` and `colorSpline`.
6. Return $vec4(color.r, color.g, color.b, alpha)$.

**Evaluating Color by Integer $isoValue$:**
1. If $isoValue < 0$ or $isoValue > 255$, return $vec4(-1)$.
2. Otherwise, return `indexedTransferFunction[isoValue]`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class CubicSplineTFWithIndexedCaching:
    def __init__(self):
        self.color_points = []  # List of (iso_value: int, color: vec3)
        self.alpha_points = []  # List of (iso_value: int, alpha: float)
        self.color_splines = []
        self.alpha_splines = []
        self.indexed_tf = [vec4(0,0,0,0)] * 256
        self.color_mapping_texture = None
        self.update_color_texture = False
        self.threshold = (0, 255)
        
        # Initialization
        self.add_color_point(vec3(1,1,1), 0)
        self.add_color_point(vec3(1,1,1), 255)
        self.add_alpha_point(1.0, 0)
        self.add_alpha_point(1.0, 255)

    def add_color_point(self, color, iso_value):
        if iso_value <= 0 or iso_value >= self.color_points[-1][0] or len(self.color_points) > 255:
            return
        # Insert maintaining sorted order
        self.color_points.append((iso_value, color))
        self.color_points.sort(key=lambda p: p[0])
        self.update_function()

    def add_alpha_point(self, alpha, iso_value):
        if iso_value <= 0 or iso_value >= self.alpha_points[-1][0] or len(self.alpha_points) > 255:
            return
        # Insert maintaining sorted order
        self.alpha_points.append((iso_value, alpha))
        self.alpha_points.sort(key=lambda p: p[0])
        self.update_function()

    def remove_color_point(self, index):
        if index <= 0 or index >= len(self.color_points):
            return
        self.color_points.pop(index)
        self.update_function()

    def remove_alpha_point(self, index):
        if index <= 0 or index >= len(self.alpha_points):
            return
        self.alpha_points.pop(index)
        self.update_function()

    def set_alpha(self, index, alpha):
        if index < 0 or index >= len(self.alpha_points):
            return
        self.alpha_points[index] = (self.alpha_points[index][0], clamp(alpha, 0.0, 1.0))
        self.update_function()

    def set_threshold(self, min_iso, max_iso):
        min_iso = clamp(min_iso, 0, max_iso - 1)
        max_iso = clamp(max_iso, min_iso + 1, 255)
        self.threshold = (min_iso, max_iso)

    def set_color(self, index, color):
        if index < 0 or index >= len(self.color_points):
            return
        self.color_points[index] = (self.color_points[index][0], clamp(color, vec3(0), vec3(1)))
        self.update_function()

    def set_alpha_point_iso_value(self, index, iso_value):
        if index <= 0 or index >= len(self.alpha_points):
            return
        if iso_value <= 0 or iso_value >= 255:
            return
        self.alpha_points[index] = (iso_value, self.alpha_points[index][1])
        if not is_sorted(self.alpha_points, key=lambda p: p[0]):
            self.alpha_points.sort(key=lambda p: p[0])
        self.update_function()

    def set_color_point_iso_value(self, index, iso_value):
        if index <= 0 or index >= len(self.color_points):
            return
        if iso_value <= 0 or iso_value >= 255:
            return
        self.color_points[index] = (iso_value, self.color_points[index][1])
        if not is_sorted(self.color_points, key=lambda p: p[0]):
            self.color_points.sort(key=lambda p: p[0])
        self.update_function()

    def update_function(self):
        # Rebuild splines from points
        self.color_splines = build_cubic_splines(self.color_points)
        self.alpha_splines = build_cubic_splines(self.alpha_points)
        
        # Update indexed cache
        for i in range(256):
            self.indexed_tf[i] = self._evaluate_float(i / 255.0)
        
        self.update_color_texture = True

    def _evaluate_float(self, t):
        iso_value = int(255.0 * t)
        
        # Find alpha interval
        current_alpha_point = None
        next_alpha_point = None
        for i in range(len(self.alpha_points) - 1):
            if self.alpha_points[i][0] <= iso_value <= self.alpha_points[i+1][0]:
                current_alpha_point = self.alpha_points[i]
                next_alpha_point = self.alpha_points[i+1]
                break
        
        if current_alpha_point is None:
            alpha = 0.0
        else:
            eval_at = (iso_value - current_alpha_point[0]) / (next_alpha_point[0] - current_alpha_point[0])
            alpha = self.alpha_splines[i].get_point_on_spline(eval_at).r
            
        # Find color interval
        current_color_point = None
        next_color_point = None
        for i in range(len(self.color_points) - 1):
            if self.color_points[i][0] <= iso_value <= self.color_points[i+1][0]:
                current_color_point = self.color_points[i]
                next_color_point = self.color_points[i+1]
                break
                
        if current_color_point is None:
            color = vec3(0)
        else:
            eval_at = (iso_value - current_color_point[0]) / (next_color_point[0] - current_color_point[0])
            color = self.color_splines[i].get_point_on_spline(eval_at)
            
        return vec4(color.r, color.g, color.b, alpha)

    def evaluate_color_float(self, t):
        return self._evaluate_float(t)

    def evaluate_color_int(self, iso_value):
        if iso_value < 0 or iso_value > 255:
            return vec4(-1)
        return self.indexed_tf[iso_value]

    def update_texture(self):
        if self.color_mapping_texture is None:
            self.color_mapping_texture = create_1d_texture(
                format=GL_RGBA, internal_format=GL_RGBA, 
                data_type=GL_FLOAT, size=256, 
                data=self.indexed_tf.data(),
                min_filter=GL_LINEAR, mag_filter=GL_LINEAR, 
                wrap=GL_CLAMP_TO_EDGE
            )
        elif self.update_color_texture:
            update_texture_data(self.color_mapping_texture, 
                                data=self.indexed_tf.data(), 
                                format=GL_RGBA, data_type=GL_FLOAT, size=256)
            self.update_color_texture = False
```

**Edge Cases and Pitfalls:**
- **Insertion Bounds:** New points must have `isoValue > 0` and `isoValue <` the last point's iso value. Size must be $\le 255$ [S1, unofficial].
- **Removal Bounds:** Index must be $> 0$ and $<$ size [S1, unofficial].
- **Threshold Bounds:** Min must be $<$ max, max must be $\le 255$ [S1, unofficial].
- **Iso Value Modification Bounds:** Iso value must be $> 0$ and $< 255$ [S1, unofficial].
- **Precision:** Float evaluation converts to integer index, potentially losing sub-integer precision compared to direct spline evaluation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Evaluation by integer iso value is $O(1)$ due to indexed caching. Evaluation by float $t$ is $O(N)$ where $N$ is the number of control points, due to interval search. Updating the cache is $O(256 \times N)$.
- **Memory Complexity:** $O(N)$ for control points and splines, plus $O(1)$ fixed size for the 256-entry indexed array and texture.
- **Measured Performance:** The indexed cache provides fast access for rendering loops, while the spline ensures smooth interpolation for UI or non-integer queries [S1, unofficial].
- **Strengths:** Fast integer lookup, smooth interpolation, lazy texture updates reduce GPU overhead.
- **Limitations:** Fixed 256-entry resolution limits precision for float inputs. Sorting overhead when modifying iso values.
- **When to Use:** When fast, discrete lookup is needed for volume rendering, and smooth interpolation is required for UI or parameter tweaking.
- **When Not to Use:** When higher resolution than 256 is needed, or when dynamic resolution is required.

## Background
This implementation originates from the `VolumeRendering` project by Jose Villegas, providing a practical class for managing 1D transfer functions in Cinder-based volume rendering applications [S1, unofficial]. It extends the theoretical cubic spline transfer function by adding practical caching and texture management for real-time rendering.

## Sources
[S1] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
