# Data Slicing / Interval Set Rendering

Also called: Fast Volume Rendering of Data Slices, Interval Volume Slicing, Real-time Data Slicing / Interval Set Rendering, Real-time Slicing of Data Space, Splatting of Interval Sets

## Overview
Data Slicing is an algorithm for the rapid selection and representation of scalar field data on regular grids, enabling fast interactive cycling through data space by selecting and rendering specific value intervals [S1]. It solves the problem of real-time visualization of large, complex 3D scalar fields by avoiding the computational cost of back-to-front sorting required by traditional volume rendering [S1]. Unlike its siblings, it relies on a pre-sorted linear list of data values to filter points near a contour or interval, rendering them with a constant color which eliminates the need for depth sorting [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field defined on a regular grid (or unstructured grid with limitations) [S1].
*   **Output:** A 2D image representing the interval set surrounding a specified contour value or range [S1].

**Data Structures:**
The core data structure is a sorted linear list of selected data points, referred to as `SoListSplat` [S1].
*   **Layout:** An array of four-component vectors (e.g., `vec4`) [S1].
    *   Components 0-2: 3D coordinates $(x, y, z)$ of the grid point [S1].
    *   Component 3: The scalar data value at that point [S1].
*   **Metadata:** The structure includes members `bboxMin` and `bboxMax` storing the coordinate boundaries of the data [S1].
*   **Sorting:** The list is sorted by the scalar data value (component 3) using a quicksort algorithm (`qsort`) [S1].

**Parameters:**
*   **User Min / User Max:** The lower and upper bounds of the data value range of interest [S1]. Values outside this range are discarded during list construction [S1].
*   **Tolerance:** The distance from a specific contour value within which data points are selected [S1].
*   **Delta:** The increment step size used when cycling through interval sets during interactive exploration [S1]. Typical values are tuned to avoid sudden visual jumps; very small increments reveal fine-grained changes in point density [S1].

**Algorithm Steps:**
1.  **Initialization:** Define the selection range `[User Min, User Max]` and the cycling increment `delta` [S1].
2.  **Filtering and Sorting:**
    *   Traverse the entire scalar field dataset once [S1].
    *   Extract grid points whose scalar values fall within `[User Min, User Max]` [S1].
    *   Store these points in a list where each entry contains $(x, y, z, \text{value})$ [S1].
    *   Sort the list by the scalar value component using quicksort [S1].
    *   *Note:* This step provides substantial data compression, often reducing the point count to less than 10% of the original set [S1].
3.  **Selection (Binary Search):**
    *   Given a target contour value $C$, determine the subset of data points within a specified tolerance $\epsilon$ of $C$ [S1].
    *   Use binary search on the sorted list to find the indices corresponding to values in $[C - \epsilon, C + \epsilon]$ [S1].
4.  **Rendering (Splatting):**
    *   Splat the selected data points to the screen using hardware-assisted rasterization [S1].
    *   **Primitive:** Use point splats (OpenGL point sizes > 10 pixels) or texture splats [S1]. Point splats are significantly faster than texture splats [S1].
    *   **Color/Opacity:** Assign a constant color and opacity to all splats in the interval set [S1].
    *   **Compositing:** Because the color is homogeneous, the order of rendering is immaterial; no back-to-front or front-to-back sorting is required [S1].
    *   **Depth Handling:** When combining with other volume rendering passes, perform volume rendering last while testing but not setting the z-buffer during splatting to allow depth blending [S1].
5.  **Animation/Exploration:**
    *   Cycle through the data space by adjusting the range `[User Min, User Max]` or shifting the contour value $C$ by `delta` [S1].
    *   Re-run steps 3-4 for the new interval [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import struct

class DataPoint:
    """Represents a single data point in the sorted list."""
    def __init__(self, x, y, z, value):
        self.x = x
        self.y = y
        self.z = z
        self.value = value

    def __lt__(self, other):
        return self.value < other.value

class IntervalSetRenderer:
    def __init__(self, grid_data):
        """
        Initialize the renderer.
        grid_data: List of tuples (x, y, z, value) representing the raw scalar field.
        """
        self.grid_data = grid_data
        self.sorted_list = []
        self.bbox_min = None
        self.bbox_max = None

    def build_sorted_list(self, user_min, user_max):
        """
        Step 2: Filter and sort the data.
        """
        filtered_points = []
        for (x, y, z, value) in self.grid_data:
            if user_min <= value <= user_max:
                filtered_points.append(DataPoint(x, y, z, value))
        
        # Sort by value using quicksort logic (simulated by Python's sort)
        filtered_points.sort()
        
        self.sorted_list = filtered_points
        
        # Compute bounding box
        if filtered_points:
            xs = [p.x for p in filtered_points]
            ys = [p.y for p in filtered_points]
            zs = [p.z for p in filtered_points]
            self.bbox_min = (min(xs), min(ys), min(zs))
            self.bbox_max = (max(xs), max(ys), max(zs))
        else:
            self.bbox_min = (0, 0, 0)
            self.bbox_max = (0, 0, 0)

    def get_interval_points(self, contour_value, tolerance):
        """
        Step 3: Select points near the contour value using binary search.
        """
        lower_bound = contour_value - tolerance
        upper_bound = contour_value + tolerance
        
        # Binary search for the start index
        low = 0
        high = len(self.sorted_list)
        start_idx = high
        
        while low < high:
            mid = (low + high) // 2
            if self.sorted_list[mid].value < lower_bound:
                low = mid + 1
            else:
                high = mid
                start_idx = mid
        
        # Binary search for the end index
        low = start_idx
        high = len(self.sorted_list)
        end_idx = len(self.sorted_list)
        
        while low < high:
            mid = (low + high) // 2
            if self.sorted_list[mid].value <= upper_bound:
                low = mid + 1
            else:
                high = mid
                end_idx = mid
                
        return self.sorted_list[start_idx:end_idx]

    def render(self, points, camera_matrix, projection_matrix, viewport_width, viewport_height):
        """
        Step 4: Splat the points to the screen.
        Note: This is a conceptual representation of the rasterization step.
        Actual implementation relies on GPU hardware (OpenGL/DirectX).
        """
        # 1. Transform points to clip space
        clip_points = []
        for p in points:
            # Apply MVP matrix to homogeneous coordinate (x, y, z, 1)
            # This step is handled by the graphics pipeline in practice
            clip_points.append(self.transform_point(p, camera_matrix, projection_matrix))
            
        # 2. Rasterize
        # For each point, determine its screen footprint
        # Use point splats with size > 10 pixels for performance
        # Assign constant color C and opacity O
        # Draw points to framebuffer
        # Since color is constant, no depth sorting is needed
        pass

    def transform_point(self, p, cam_mat, proj_mat):
        # Simplified MVP transformation
        # In reality, this is done by the GPU vertex shader
        return (p.x, p.y, p.z, 1.0) # Placeholder for actual matrix multiplication

    def cycle_interval(self, current_value, delta):
        """
        Step 5: Animation loop.
        """
        new_value = current_value + delta
        return new_value

# Example Usage
# renderer = IntervalSetRenderer(raw_grid_data)
# renderer.build_sorted_list(user_min=0.0, user_max=1.0)
# points = renderer.get_interval_points(contour_value=0.5, tolerance=0.05)
# renderer.render(points, cam, proj, w, h)
```

**Edge Cases and Pitfalls:**
*   **High-Frequency Data:** For data varying widely or containing high frequencies, the sorted list approach is less advantageous than octree-based skipping methods, as the compression ratio may be poor [S1].
*   **Visual Popping:** Sudden visual "popping" occurs when cycling through intervals because grid point density can change abruptly at certain scalar values [S1]. Mitigation: redefine the range and reduce `delta` to a very small increment to smoothly traverse the transition [S1].
*   **Discrete Representation:** The algorithm operates in discrete space rather than continuous reconstruction. Areas of high gradient may be missed when intervals are small. It does not strictly represent true contour surfaces or interval volumes, but rather a slice plane through data space [S1].
*   **Texture Splat Performance:** Texture splats are significantly slower than point splats. Optimization of texture splat performance is noted as a future goal [S1].
*   **Unstructured Grids:** The splatting algorithm used here is not accurate for curvilinear and unstructured data sets; it represents only the raw data points [S1].

## Operating characteristics
*   **Time Complexity:** The initial sorting step relies on quicksort ($O(N \log N)$) [S1]. Subsequent rendering steps are $O(K)$, where $K$ is the number of selected points in the interval [S1].
*   **Memory Complexity:** Memory usage is reduced via substantial data compression by storing only interval-selected points [S1]. The sorted list stores only points within the active interval [S1].
*   **Measured Performance:** On an SGI Onyx/VTX (R4400 CPU @ 100 MHz), the Global Climate model (320x160x19 mesh, ~1M points) renders textured splats at slightly over 1 frame per second [S1]. Other datasets (HEPIPH, Post, SOD, Shockwave) achieve real-time performance [S1].
*   **Compression Ratios:** Vary by dataset. Examples from [S1]:
    *   HEPIPH: 262,144 -> 29,111 points (11%)
    *   Climate: 718,911 -> 67,667 points (9.4%)
    *   Post: 972,800 -> 94,798 points (9.7%)
    *   SOD: 16,096 -> 1,498 points (9.3%)
    *   Shockwave: 1,091,444 -> 12,976 points (1.2%)
    *   Sod: 1,728,000 -> 8,356 points (0.5%)
*   **Strengths:** Allows animation through even the largest data sets because the slice limits the number of data points to a very small fraction (one-hundredth or less) of the total set [S1]. No back-to-front sorting is required [S1].
*   **Limitations:** Does not handle high-frequency data well due to poor compression [S1]. Not accurate for unstructured grids [S1]. Texture splats are slow [S1].
*   **When to use:** Applications with high demands for interactivity, such as virtual reality settings, or for off-loading reasonable data sets from very large simulations on massively parallel supercomputers for quick overview [S1].
*   **When not to use:** When true contour surface accuracy is required, or when dealing with high-frequency scalar data where isosurfaces fail but precise interval representation is needed [S1].

## Background
*   **Origin:** The method was introduced by Roger Crawford at Lawrence Livermore National Laboratory [S1].
*   **Paper:** "Real-time slicing of data space" [S1].
*   **History:** The algorithm draws upon three main ideas: a fast indexing scheme (sorted list), hardware-assisted splatting, and the observation from Max et al. that homogeneous color rendering requires no depth sorting [S1]. It was designed to address the problematic nature of real-time rendering of iso-contour surfaces for large complex data sets [S1].
*   **Usage:** Applied to Global Climate model (percent cloudiness), HEPA filter flow magnitude, proton density, vorticity in fluid flow past a post, and astrophysics shock wave data [S1].

## Sources
[S1] Real-time slicing of data space, https://www.osti.gov/biblio/632502, fetched 2026-10-26
