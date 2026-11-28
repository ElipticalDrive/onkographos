# VisC2D Color Coding

Also called: 2D Palette Mapping

## Overview
VisC2D Color Coding is a semantic label-based visualization method that maps scalar physical data (specifically pressure) to a fixed 256-color grayscale palette stored in Device Independent Bitmap (DIB) format [S1, unofficial]. It solves the problem of visualizing explosion shock wave propagation by explicitly classifying distinct physical media (wall, air, shock wave) and assigning them unique color indices, where brightness correlates linearly with pressure magnitude [S1, unofficial]. It differs from general transfer function methods by using hard classification thresholds for static media and a fixed linear segment mapping for the variable medium, rather than continuous opacity/gradient mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A dataset of applicable scalar data (pressure values $P$) defined on a grid or cell structure.
*   Output: A DIB bitmap image where each pixel corresponds to a data point, indexed by a color from a 256-entry palette.

**Data Structures:**
*   **Palette:** An array of 256 colors.
    *   Index 0: Color for "Wall".
    *   Index 1: Color for "Air".
    *   Indices 2–255: Grayscale colors ordered by brightness (Index 2 is darkest gray, Index 255 is brightest white/gray).
*   **Image Buffer:** A DIB bits array matching the spatial dimensions of the input data.

**Parameters:**
*   Palette Size: 256 colors [S1, unofficial].
*   Wall Pressure Constant: $0$ [S1, unofficial].
*   Air Pressure Constant: $1.087$ [S1, unofficial].

**Steps:**
1.  **Classification:** Iterate through each data point $P$ to classify it into one of three categories: Wall, Air, or Shock Wave.
    *   If $P = 0$, classify as Wall.
    *   If $P = 1.087$, classify as Air.
    *   Otherwise, classify as Shock Wave.
2.  **Index Assignment:**
    *   If classified as Wall, assign color index $0$.
    *   If classified as Air, assign color index $1$.
    *   If classified as Shock Wave, calculate the color index $I$ using the linear mapping formula below.
3.  **Image Generation:** Write the assigned color index to the corresponding pixel in the DIB bitmap.

**Formula:**
For shock wave data, the color index $I$ is calculated as:
$$ I = \frac{253(P - P_{\min})}{P_{\max} - P_{\min}} + 2 $$
Where:
*   $P$ is the current pressure value.
*   $P_{\min}$ is the minimum pressure value in the entire dataset.
*   $P_{\max}$ is the maximum pressure value in the entire dataset.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def visc2d_color_coding(pressure_data, p_min, p_max):
    """
    Maps pressure data to 256-color DIB indices.
    pressure_data: 2D array or list of pressure values.
    p_min: float, minimum pressure in dataset.
    p_max: float, maximum pressure in dataset.
    Returns: 2D array of color indices (0-255).
    """
    # Constants defined in source
    WALL_PRESSURE = 0.0
    AIR_PRESSURE = 1.087
    
    # Initialize output buffer
    num_rows = len(pressure_data)
    num_cols = len(pressure_data[0])
    indices = [[0] * num_cols for _ in range(num_rows)]
    
    # Pre-calculate denominator to avoid division by zero or repeated calc
    # Note: If P_max == P_min, the source does not specify behavior. 
    # Standard practice implies uniform color, but formula yields division by zero.
    # [G] Guard added for implementation safety.
    if p_max == p_min:
        denom = 1.0
    else:
        denom = p_max - p_min
        
    for r in range(num_rows):
        for c in range(num_cols):
            p = pressure_data[r][c]
            
            # Step 1: Classification
            if p == WALL_PRESSURE:
                indices[r][c] = 0
            elif p == AIR_PRESSURE:
                indices[r][c] = 1
            else:
                # Step 2: Shock Wave Mapping
                # Formula: 253(P - P_min)/(P_max - P_min) + 2
                # Clamp result to integer range [2, 255] if necessary due to precision
                raw_index = (253 * (p - p_min) / denom) + 2
                # [G] Cast to integer and clamp to valid palette range
                idx = int(round(raw_index))
                if idx < 2:
                    idx = 2
                elif idx > 255:
                    idx = 255
                indices[r][c] = idx
                
    return indices
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for $P=0$ and $P=1.087$ may fail if data is not perfectly constant. [G] In practice, a small epsilon tolerance should be used for classification checks.
*   **Division by Zero:** If $P_{\max} = P_{\min}$, the denominator is zero. The source does not specify handling. [G] Implementation should handle this case (e.g., assign a default index or skip mapping).
*   **Boundary Conditions:** The formula maps $P_{\min}$ to index 2 and $P_{\max}$ to index 255. Values outside $[P_{\min}, P_{\max}]$ will map outside the intended range. [G] Clamping is required.

## Subtopics

### VisC2D Mapping
VisC2D Mapping is the specific application of the color coding method to explosion shock wave visualization. It is distinguished by its explicit handling of constant-pressure media (wall at 0, air at 1.087) and the use of a 254-segment linear partition for the shock wave pressure data [S1, unofficial]. It is chosen when the goal is to explicitly observe the transmission of shock waves in time-order frame animations using simple grayscale intensity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of data points, as each point is processed once with constant-time arithmetic.
*   **Memory Complexity:** $O(N)$ for the output DIB bitmap and palette storage.
*   **Performance:** The method is computationally lightweight, suitable for generating frame animations of explosive fields [S1, unofficial].
*   **Strengths:** Device independence via DIB format; simple grayscale mapping makes pressure gradients easy to interpret visually; explicit classification prevents ambiguity between static media and dynamic shock waves [S1, unofficial].
*   **Limitations:** Limited to 256 colors; assumes constant values for wall and air which may not hold in all physical models; linear mapping may not highlight subtle pressure variations if the range $[P_{\min}, P_{\max}]$ is large [S1, unofficial].
*   **When to use:** Visualization of explosion shock waves where distinct media separation and pressure magnitude are key.
*   **When not to use:** When continuous opacity or multi-dimensional data (e.g., temperature + pressure) needs to be visualized simultaneously.

## Background
The method was introduced in a 2006 paper by Chinese researchers, focusing on visualization techniques for explosion phenomena [S1, unofficial]. It utilizes Windows API functions for displaying the resulting DIB bitmaps as frame animations to observe shock wave transmission [S1, unofficial]. The approach is part of a broader pipeline involving data normalization and cell division for applicable data extraction [S1, unofficial].

## Sources
[S1] PDF A09-65-A-_중국-지불됨-Accepted_ 1125-Visualization_Technique_of_Explosion ..., http://paper.ijcsns.org/07_book/200611/200611A09.pdf, fetched 2026-10-29 (unofficial)
