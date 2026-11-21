# Midpoint Displacement Fractal

Also called: Plasma fractal grid generation

## Overview
Midpoint Displacement Fractal generates a 2D noise grid using recursive midpoint displacement to create a fractal terrain or noise map [S1, unofficial]. It differs from its parent, Random Heightmap Terrain Generation, by imposing spatial correlation through recursive averaging and roughening, rather than assigning independent random heights to all vertices [S1, unofficial]. This method is chosen when a continuous, fractal-like heightmap is required for procedural voxel generation.

## How it works
**Inputs:**
- `width` (int): Width of the output grid.
- `height` (int): Height of the output grid.
- `roughness` (float): Controls the fractal roughness.

**Outputs:**
- `grid_`: A 2D float array of size `[width][height]` containing the generated height values.

**Data Structures:**
- `grid_[width][height]`: Stores the height value for each cell. Values are typically normalized.
- `roughness_`: Internal parameter, calculated as `roughness / width`.

**Algorithm Steps:**

1.  **Initialization**:
    - Calculate `roughness_ = roughness / width` [S1, unofficial].
    - Allocate `grid_[width][height]` [S1, unofficial].
    - Set the random number generator seed [S1, unofficial].
    - Set the four corner points to random values in the range `[-0.5, 0.5]`:
      - `grid_[0][0]`
      - `grid_[0][height-1]`
      - `grid_[width-1][0]`
      - `grid_[width-1][height-1]`
    - Call `generate(0, 0, width-1, height-1)` [S1, unofficial].

2.  **Recursive Generation (`generate(xl, yl, xh, yh)`)**:
    - Compute midpoints:
      - `xm = (xl + xh) / 2`
      - `ym = (yl + yh) / 2`
    - **Base Case**: If `xl == xm` and `yl == ym`, return [S1, unofficial].
    - **Edge Midpoints**: Compute the unroughened midpoints of the four edges by averaging the adjacent corners:
      - `grid_[xm][yl] = 0.5f * (grid_[xl][yl] + grid_[xh][yl])`
      - `grid_[xm][yh] = 0.5f * (grid_[xl][yh] + grid_[xh][yh])`
      - `grid_[xl][ym] = 0.5f * (grid_[xl][yl] + grid_[xl][yh])`
      - `grid_[xh][ym] = 0.5f * (grid_[xh][yl] + grid_[xh][yh])`
    - **Center Point**: Compute the center point by averaging the two horizontal edge midpoints (or vertical, equivalent):
      - `center_val = 0.5f * (grid_[xm][yl] + grid_[xm][yh])`
      - Roughen the center value: `grid_[xm][ym] = roughen(center_val, xl, xh)` [S1, unofficial].
    - **Roughen Edges**: Roughen the four edge midpoints using `roughen()` with appropriate span parameters:
      - `grid_[xm][yl] = roughen(grid_[xm][yl], xl, xh)`
      - `grid_[xm][yh] = roughen(grid_[xm][yh], xl, xh)`
      - `grid_[xl][ym] = roughen(grid_[xl][ym], yl, yh)`
      - `grid_[xh][ym] = roughen(grid_[xh][ym], yl, yh)`
    - **Recurse**: Call `generate()` on the four sub-quadrants:
      - `generate(xl, yl, xm, ym)`
      - `generate(xm, yl, xh, ym)`
      - `generate(xl, ym, xm, yh)`
      - `generate(xm, ym, xh, yh)`

3.  **Roughening Function (`roughen(v, l, h)`)**:
    - Generate Gaussian noise `n` with mean 0 and standard deviation proportional to the span `(h - l)` and `roughness_` [S1, unofficial].
    - Return `v + n` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

class MidpointDisplacement:
    def __init__(self, width, height, roughness):
        self.width = width
        self.height = height
        self.roughness_ = roughness / width
        self.grid = [[0.0 for _ in range(height)] for _ in range(width)]
        self._initialize()

    def _initialize(self):
        # Set corners to random values in [-0.5, 0.5]
        self.grid[0][0] = random.uniform(-0.5, 0.5)
        self.grid[0][self.height - 1] = random.uniform(-0.5, 0.5)
        self.grid[self.width - 1][0] = random.uniform(-0.5, 0.5)
        self.grid[self.width - 1][self.height - 1] = random.uniform(-0.5, 0.5)
        self._generate(0, 0, self.width - 1, self.height - 1)

    def _generate(self, xl, yl, xh, yh):
        xm = (xl + xh) // 2
        ym = (yl + yh) // 2

        # Base case: if the quadrant is a single point, stop
        if xl == xm and yl == ym:
            return

        # Compute unroughened edge midpoints
        # Bottom edge
        self.grid[xm][yl] = 0.5 * (self.grid[xl][yl] + self.grid[xh][yl])
        # Top edge
        self.grid[xm][yh] = 0.5 * (self.grid[xl][yh] + self.grid[xh][yh])
        # Left edge
        self.grid[xl][ym] = 0.5 * (self.grid[xl][yl] + self.grid[xl][yh])
        # Right edge
        self.grid[xh][ym] = 0.5 * (self.grid[xh][yl] + self.grid[xh][yh])

        # Compute center point
        center_val = 0.5 * (self.grid[xm][yl] + self.grid[xm][yh])
        self.grid[xm][ym] = self._roughen(center_val, xl, xh)

        # Roughen edge midpoints
        self.grid[xm][yl] = self._roughen(self.grid[xm][yl], xl, xh)
        self.grid[xm][yh] = self._roughen(self.grid[xm][yh], xl, xh)
        self.grid[xl][ym] = self._roughen(self.grid[xl][ym], yl, yh)
        self.grid[xh][ym] = self._roughen(self.grid[xh][ym], yl, yh)

        # Recurse on four sub-quadrants
        self._generate(xl, yl, xm, ym)
        self._generate(xm, yl, xh, ym)
        self._generate(xl, ym, xm, yh)
        self._generate(xm, ym, xh, yh)

    def _roughen(self, v, l, h):
        # Span determines the magnitude of noise
        span = h - l
        # Gaussian noise scaled by roughness_ and span
        # Note: Standard deviation is typically proportional to span * roughness_
        noise = random.gauss(0, span * self.roughness_)
        return v + noise
```

**Parameters:**
- `roughness`: Float controlling fractal roughness. Typical values are not specified in the sources [S1, unofficial].
- `width`, `height`: Integers defining the grid dimensions [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Integer Division**: The midpoint calculation `(xl + xh) // 2` must handle odd/even dimensions correctly to ensure the recursion terminates. If `xl + xh` is odd, `xm` will be the floor, and the base case `xl == xm` will eventually trigger for single-point cells [S1, unofficial].
- **Precision**: Using floating-point arithmetic for grid indices is not recommended; integer arithmetic should be used for coordinates [S1, unofficial].
- **Boundary Conditions**: The algorithm assumes the grid boundaries are fixed by the initial corner values. No wrapping or periodic boundary conditions are mentioned [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N \log N)$ or $O(N)$ depending on implementation, where $N = width \times height$. Each cell is visited a constant number of times per level of recursion. The depth of recursion is $\log_2(\min(width, height))$ [S1, unofficial].
**Memory Complexity:** $O(N)$ for the grid storage [S1, unofficial].
**Performance:** Not given in the sources [S1, unofficial].
**Strengths:** Generates smooth, fractal-like terrain with minimal code.
**Limitations:** The roughness parameter is scaled by width, which may lead to inconsistent results across different grid sizes if not normalized properly [S1, unofficial].

## Background
The method is used in the context of a simple voxel framework for rendering cubes in jMonkeyEngine 3 [S1, unofficial]. It is a classic technique for generating fractal noise and terrain.

## Sources
[S1] destroflyer/cubes, https://github.com/destroflyer/cubes, fetched 2026-10-26 (unofficial)
