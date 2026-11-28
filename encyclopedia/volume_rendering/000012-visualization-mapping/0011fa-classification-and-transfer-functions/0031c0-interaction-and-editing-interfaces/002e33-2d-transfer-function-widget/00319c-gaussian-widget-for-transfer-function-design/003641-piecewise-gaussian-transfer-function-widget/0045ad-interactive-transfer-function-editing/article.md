# Interactive Transfer Function Editing

Also called: Focus Cut-away, Interactive Transfer Function Editor, Structural Cut-away, TF GUI, Transfer Function GUI Editor

## Overview
Interactive Transfer Function (TF) Editing provides a graphical user interface for defining, modifying, and managing the color and opacity mappings used in volume rendering [S3, S5, unofficial]. It solves the problem of manually specifying which data values are visible and how they are colored, allowing users to iteratively refine visualizations through direct manipulation [S3, unofficial]. Unlike static parameter tuning, this method integrates real-time feedback, history management, and structural context (such as cut-away views) into the editing workflow [S1, S5, unofficial]. It distinguishes itself from basic widget-based editing by offering persistent state management (history trees) and structural classification integration (focus+context) [S1, S5, unofficial].

## How it works
The method comprises three primary interaction paradigms: the basic GUI widget for point-based editing [S3, unofficial], the Iso-Dist UI for non-linear mapping [S4, unofficial], and the History Tree for state management [S5, unofficial].

### Inputs and Outputs
*   **Inputs:** User mouse/keyboard events (clicks, drags, wheel scrolls), current volume data statistics (min/max, histogram), and existing TF control points [S3, S4, unofficial].
*   **Outputs:** Updated transfer function texture (color/opacity maps), rendered volume visualization, and history tree state [S3, S5, unofficial].

### Data Structures
The core GUI implementation maintains the following arrays and variables [S3, unofficial]:
*   `pointList`: Array of 2D points `(x, y)` storing the position of transfer function control points.
*   `colorList`: Array of `glm::vec4` storing RGBA colors for each control point.
*   `colorPosList`: Array storing the x-positions of the color selector indicators for each point.
*   `colorPickerPosList`: Array storing the picker positions for each point.
*   `ptsCounter`: Integer tracking the number of active control points.
*   `baseColors`: Array of 6 `glm::vec4` gradient stops:
    1.  `[1.0, 0.0, 0.0, 1.0]` (Red)
    2.  `[1.0, 0.0, 1.0, 1.0]` (Magenta)
    3.  `[0.0, 0.0, 1.0, 1.0]` (Blue)
    4.  `[0.0, 1.0, 1.0, 1.0]` (Cyan)
    5.  `[0.0, 1.0, 0.0, 1.0]` (Green)
    6.  `[1.0, 1.0, 0.0, 1.0]` (Yellow)
*   `alpha`: GUI transparency factor (default 0.5, 0.7 during window drag) [S3, unofficial].
*   `DIVBASESC`: Constant `MAXWSC / 6`, used to divide the color selector into 6 segments [S3, unofficial].

### Interaction Steps

#### 1. Initialization
1.  Load textures: `transferFunction.jpg`, `point.png`, `selector.png` using an image library (e.g., DevIL) [S3, unofficial].
2.  Set window/viewport bounds (`MINW`, `MAXW`, `MINH`, `MAXH` for graph; `MINWSC`, `MAXWSC`, `MINHSC`, `MAXHSC` for color selector; `MINWPC`, `MAXWPC`, `MINHPC`, `MAXHPC` for picker) [S3, unofficial].
3.  Initialize `ptsCounter = 0`, `pointSelected = 1`, `alpha = 0.5` [S3, unofficial].
4.  Simulate initial mouse presses to establish default state and call `UpdatePallete()` [S3, unofficial].

#### 2. Mouse Interaction Logic
The interface handles mouse events based on the clicked region [S3, unofficial]:

*   **Left Press in Graph Area (`MINW` to `MAXW`, `MINH` to `MAXH`):**
    *   If no existing point is picked (`Picking` returns false) and `ptsCounter < MAXPOINT - 1`:
        1.  Create new point at `(w, h)`.
        2.  Increment `ptsCounter`.
        3.  Set `pointSelected = ptsCounter`.
        4.  Call `UpdateColorPoint()` [S3, unofficial].
    *   If an existing point is picked:
        1.  Set `indicatorSC` to the point's `colorPosList` value [S3, unofficial].
        2.  Compute `currentColor` via linear interpolation between `baseColors` based on `indicatorSC` and `DIVBASESC`:
            $$ t = \frac{\text{indicatorSC} \pmod{\text{DIVBASESC}}}{\text{DIVBASESC}} $$
            $$ \text{currentColor} = \text{baseColors}[k] \times (1.0 - t) + \text{baseColors}[(k+1) \pmod 6] \times t $$
            where $k = \lfloor \text{indicatorSC} / \text{DIVBASESC} \rfloor$ [S3, unofficial].
        3.  Update `currentColorPickerPos` from `colorPickerPosList` [S3, unofficial].
    *   Call `SortPoints()` and `UpdatePallete()` [S3, unofficial].

*   **Left Press in Color Selector Area (`MINWSC` to `MAXWSC`, `MINHSC` to `MAXHSC`):**
    1.  Set `indicatorSC = w` [S3, unofficial].
    2.  Compute `currentColor` using the same interpolation formula as above [S3, unofficial].
    3.  If `pointSelected != 0`, call `UpdateColorPoint()` [S3, unofficial].
    4.  Call `UpdatePallete()` [S3, unofficial].

*   **Left Press in Color Picker Area (`MINWPC` to `MAXWPC`, `MINHPC` to `MAXHPC`):**
    1.  Clamp `w` to `[MINWPC + 6, MAXWPC - 6]` and `h` to `[MINHPC + 6, MAXHPC - 6]` to prevent out-of-bounds [S3, unofficial].
    2.  Update `currentColorPickerPos = (w, h)` [S3, unofficial].
    3.  If `pointSelected != 0`, call `UpdateColorPoint()` [S3, unofficial].
    4.  Call `UpdatePallete()` [S3, unofficial].

*   **Right Press:**
    1.  Call `DeletePoint(w, h)` [S3, unofficial].
    2.  If a point is picked:
        *   Save `lastPicking`.
        *   If deleting the selected point, reset `pointSelected` to 0 or 1 [S3, unofficial].
        *   Remove point from `pointList`, decrement `ptsCounter` [S3, unofficial].
        *   Call `SortPoints()` and `UpdatePallete()` [S3, unofficial].

*   **Mouse Release:**
    1.  Reset `dragDrop`, `dragDropColor`, `dragDropPicker`, `dragDropWindow` to `false` [S3, unofficial].
    2.  Reset `alpha` to `0.5` [S3, unofficial].

#### 3. Rendering Loop
1.  Render TF graph as filled quads between consecutive points with `GrsfAlpha = 0.5` [S3, unofficial].
2.  Render 6-segment base color gradient [S3, unofficial].
3.  Render selector indicator [S3, unofficial].
4.  Render alpha gradient [S3, unofficial].
5.  Render point markers using `point.png` texture [S3, unofficial].
6.  Render color picker marker using `selector.png` texture [S3, unofficial].
7.  Picking is performed using OpenGL selection buffer (`glSelectBuffer`, `glRenderMode(GL_SELECT)`, `gluPickMatrix`) to determine clicked regions [S3, unofficial].

#### 4. Iso-Dist UI Specifics
*   Display bin boundaries with black lines to indicate non-uniformity [S4, unofficial].
*   Display color map as a ramp at the top and a combined color/opacity bar below [S4, unofficial].
*   Warp opacity and color functions into non-uniform data space (Iso-Dist mode) [S4, unofficial].
*   Display a third bar as a grayscale opacity mapping [S4, unofficial].
*   Allow zooming (mouse wheel) and panning (mouse wheel down + move) [S4, unofficial].
*   Vertical axis uses logarithmic scale for opacity [S4, unofficial].

#### 5. Structural Cut-away
*   Set opacity channel of Color Cluster Transfer Function (CTF) to binary values: 1 for selected focus clusters (union of user query), 0 for others [S1, unofficial].
*   Shader skips fragments with zero opacity [S1, unofficial].

#### 6. History Tree Management
*   Store current TF and rendered volume snapshot as a tree node [S5, unofficial].
*   Display tree in a separate window with snapshot previews [S5, unofficial].
*   Allow retrieval by clicking nodes, scaling/translation by dragging, and branch deletion [S5, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
point_list = []
color_list = []
color_pos_list = []
color_picker_pos_list = []
pts_counter = 0
point_selected = 1
base_colors = [
    (1.0, 0.0, 0.0, 1.0), (1.0, 0.0, 1.0, 1.0), (0.0, 0.0, 1.0, 1.0),
    (0.0, 1.0, 1.0, 1.0), (0.0, 1.0, 0.0, 1.0), (1.0, 1.0, 0.0, 1.0)
]
alpha = 0.5
divbase_sc = maxwsc / 6.0
last_picking_var = 0
drag_drop = False
drag_drop_color = False
drag_drop_picker = False
drag_drop_window = False

def init_transfer_function_gui():
    # Load textures: transferFunction.jpg, point.png, selector.png
    # Simulate initial mouse press
    update_palette()

def on_mouse_press(w, h, button, action):
    if button == 'LEFT' and action == 'PRESS':
        if is_in_graph(w, h):
            if not picking(w, h, 'GRAPH'):
                if pts_counter < MAXPOINT - 1:
                    point_list.append((w, h))
                    pts_counter += 1
                    point_selected = pts_counter
                    update_color_point()
            else:
                indicator_sc = color_pos_list[point_selected - 1]
                current_color = interpolate_color(indicator_sc)
                current_color_picker_pos = color_picker_pos_list[point_selected - 1]
            sort_points()
            update_palette()
        elif is_in_color_selector(w, h):
            indicator_sc = w
            current_color = interpolate_color(indicator_sc)
            if point_selected != 0:
                update_color_point()
            update_palette()
        elif is_in_picker(w, h):
            w = max(minwpc + 6, min(w, maxwpc - 6))
            h = max(minhpc + 6, min(h, maxhpc - 6))
            current_color_picker_pos = (w, h)
            if point_selected != 0:
                update_color_point()
            update_palette()
    elif button == 'RIGHT' and action == 'PRESS':
        delete_point(w, h)
    elif action == 'RELEASE':
        drag_drop = False
        drag_drop_color = False
        drag_drop_picker = False
        drag_drop_window = False
        alpha = 0.5

def interpolate_color(indicator_sc):
    complete_size = (indicator_sc // divbase_sc) * divbase_sc
    current_size = indicator_sc - complete_size
    t = current_size / divbase_sc
    k = complete_size // divbase_sc
    c1 = base_colors[k]
    c2 = base_colors[(k + 1) % 6]
    return (c1[0]*(1.0-t) + c2[0]*t, c1[1]*(1.0-t) + c2[1]*t, c1[2]*(1.0-t) + c2[2]*t, c1[3]*(1.0-t) + c2[3]*t)

def delete_point(w, h):
    if picking(w, h, 'GRAPH'):
        if last_picking_var != 1 and last_picking_var != pts_counter:
            if point_selected == last_picking_var:
                point_selected = 0
            else:
                point_selected = point_selected
            sort_points(last_picking_var - 1)
            update_palette()
            point_selected = 1
            last_picking_var = -1
            pts_counter -= 1
        else:
            point_selected = 0
        drag_drop = False
        return True
    return False

def render():
    # Render graph quads with GrsfAlpha = 0.5
    # Render base color gradient
    # Render selector indicator
    # Render alpha gradient
    # Render point markers
    # Render color picker marker
    pass
```

## Subtopics

### Iso-Dist UI Interaction
A variant that warps opacity and color functions into non-uniform data space (Iso-Dist mode) to handle high dynamic range data [S4, unofficial]. It displays bin boundaries, uses a logarithmic opacity scale, and allows zooming/panning for detail transition [S4, unofficial]. Choose this when dealing with non-uniform data distributions where linear mapping flattens the histogram [S4, unofficial].

### Transfer Function History Tree
A persistent history tree that stores TFs and rendered snapshots as nodes [S5, unofficial]. It allows retrieval, scaling, translation, and branch deletion [S5, unofficial]. Choose this when reproducibility and iterative exploration of multiple TF configurations are required [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of control points and sources [S2][S3, unofficial]. For the Mixture Graph integration, CPU performance is limited by the number of nodes, while GPU performance is limited by synchronization [S2].
*   **Measured Performance:**
    *   Hippocampus (353 sources, depth 11): 22.7ms CPU, 3.5ms GPU [S2].
    *   Neocortex (1,182 sources, depth 22): 129.7ms CPU, 11.3ms GPU [S2].
    *   Polymer (15,917 sources, depth 26): 57.0ms CPU, 12.1ms GPU [S2].
*   **Memory Complexity:** Proportional to the number of control points and the size of the transfer function texture [S3, unofficial].
*   **Strengths:** Interactive, real-time feedback; supports complex structural cut-aways; persistent history [S1, S3, S5, unofficial].
*   **Limitations:** CPU version scales poorly with node count; GPU version requires synchronization overhead [S2].
*   **When to use:** Real-time volume exploration requiring manual TF adjustment [S3, unofficial].
*   **When not to use:** Automated classification without user intervention [S1, unofficial].

## Background
The method originated from implementations in C++ using OpenGL/GLFW/GLUT for interface and rendering [S3, S5, unofficial]. The Iso-Dist UI was presented by Schulze in 2006 for high dynamic range data [S4, unofficial]. The Mixture Graph integration for interactive editing was published in 2020 [S2]. The history tree approach was developed in the context of volumetric visualization tools [S5, unofficial].

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
[S2] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S3] andresz1/transfer-function-glfw3, https://github.com/andresz1/transfer-function-glfw3, fetched 2026-10-27 (unofficial)
[S4] A User Interface for High Dynamic Range Transfer ..., https://cseweb.ucsd.edu/~jschulze/publications/Schulze2006-poster.pdf, fetched 2026-10-29 (unofficial)
[S5] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
