# Image Space Outline Generation

Also called: Edge extraction from depth/normal maps, Hertzmann's outline generation, Outline Post Processing

## Overview
Image Space Outline Generation is a post-processing technique that renders non-photorealistic outlines by detecting discontinuities in screen-space depth or normal buffers [S1, S2, unofficial]. It solves the problem of defining object boundaries in volume rendering without the computational overhead of analytical geometry processing [S1, unofficial]. Unlike sibling methods that compute contours from volumetric gradients or topological features, this method relies on simple pixel-wise comparisons of existing rasterization buffers, making it significantly faster and easier to implement [G].

## How it works
**Inputs:**
*   Depth buffer containing the depth value $d(x,y)$ for each pixel $(x,y)$ [S1, unofficial].
*   Normal buffer (optional) containing the surface normal vector $\mathbf{n}(x,y)$ for each pixel [S1, unofficial].
*   Threshold parameters for edge detection [G].

**Outputs:**
*   Binary edge map or stylized outline image [G].

**Data Structures:**
*   The depth and normal buffers are stored as 2D arrays [G]. Depth values are typically stored as floating-point numbers [G]. Normal vectors are 3-component vectors [G].

**Steps:**
1.  **Depth Edge Extraction:** Iterate over each pixel $(x,y)$ and its neighbors. Apply an edge detection mechanism, such as the Sobel filter, to the depth buffer [S1, unofficial]. Depth differences are minimal between adjacent pixels of a single object and large between objects [S1, unofficial].
    $$ E_d(x,y) = \begin{cases} 1 & \text{if } |d(x,y) - d(x',y')| > \Delta d_{th} \\ 0 & \text{otherwise} \end{cases} $$
    where $(x',y')$ is a neighbor pixel and $\Delta d_{th}$ is the depth threshold [G].
2.  **Normal Edge Extraction (Optional):** Apply the same edge detection mechanism to the normal buffer [S1, unofficial]. Normals vary smoothly across a single surface but show a discontinuity between surfaces [S1, unofficial].
    $$ E_n(x,y) = \begin{cases} 1 & \text{if } \mathbf{n}(x,y) \cdot \mathbf{n}(x',y') < \theta_{th} \\ 0 & \text{otherwise} \end{cases} $$
    where $\theta_{th}$ is the cosine of the angle threshold [G].
3.  **Combination:** Combine the detected depth and normal edges to yield all perceptually significant silhouettes [S1, unofficial]. Depth edges reflect C0 discontinuities, while normal edges reflect C1 discontinuities [S1, unofficial].
    $$ E_{final}(x,y) = E_d(x,y) \lor E_n(x,y) $$
4.  **Analytical Curve Fitting (Optional):** For further processing, such as modifying line qualities, calculate analytical curves from the detected edge pixel lists [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_outlines(depth_buffer, normal_buffer=None, depth_threshold=0.0, normal_threshold=0.0):
    """
    Generate outlines from depth and normal buffers.
    
    Args:
        depth_buffer: 2D list of depth values.
        normal_buffer: 2D list of normal vectors (optional).
        depth_threshold: Minimum depth difference to be considered an edge.
        normal_threshold: Cosine threshold for normal angle difference.
        
    Returns:
        edge_map: 2D list of booleans.
    """
    height = len(depth_buffer)
    width = len(depth_buffer[0])
    edge_map = [[False for _ in range(width)] for _ in range(height)]
    offsets = [(-1, 0), (1, 0), (0, -1), (0, 1)]
    
    for y in range(height):
        for x in range(width):
            is_edge = False
            
            # Check depth discontinuities (C0)
            for dy, dx in offsets:
                ny, nx = y + dy, x + dx
                if 0 <= ny < height and 0 <= nx < width:
                    if abs(depth_buffer[y][x] - depth_buffer[ny][nx]) > depth_threshold:
                        is_edge = True
                        break
            
            # Check normal discontinuities (C1) if normal buffer is provided
            if not is_edge and normal_buffer is not None:
                for dy, dx in offsets:
                    ny, nx = y + dy, x + dx
                    if 0 <= ny < height and 0 <= nx < width:
                        cur_vec = normal_buffer[y][x]
                        nx_vec = normal_buffer[ny][nx]
                        dot_product = cur_vec[0]*nx_vec[0] + cur_vec[1]*nx_vec[1] + cur_vec[2]*nx_vec[2]
                        if dot_product < normal_threshold:
                            is_edge = True
                            break
            
            edge_map[y][x] = is_edge
            
    return edge_map
```

**Parameters:**
*   `depth_threshold`: Not given in the sources. [G]
*   `normal_threshold`: Not given in the sources. [G]

**Edge Cases:**
*   **Precision:** Depth buffer precision can cause false edges in distant scenes or true edges in close-up scenes. [G] Use logarithmic depth buffers or higher precision formats if necessary.
*   **Boundaries:** Pixels at the image border have fewer neighbors. Handle by ignoring out-of-bounds neighbors or padding the buffer. [G]
*   **Noise:** Depth and normal buffers may contain noise from rasterization or volume rendering. Filtering (e.g., Sobel, Gaussian) is recommended [S1, unofficial].

## Subtopics

### [Close-Up Image Depth Map Generation](13992-close-up-image-depth-map-generation.md)
A method to generate a Cumulative Gradient Depth Map (CGDM) for close-up images (defined as scenes with no or multiple vanishing points) by leveraging occlusion cues and local edge histograms. It distinguishes itself from [Perspective View Depth Map Generation](14031-perspective-view-depth-map-generation.md) by relying on local edge density rather than global geometric convergence, and from [Distant View Depth Map Generation](distant-view-depth-map-generation.md) by handling moderate-distance scenes with perspective distortion rather than infinite-distance horizons.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves a single pass over the buffer with constant-time neighbor checks. [G]
*   **Memory Complexity:** $O(N)$ for storing the edge map, plus the cost of the depth and normal buffers. [G]
*   **Performance:** Very fast and simple to implement, suitable for real-time applications [S1, S2, unofficial].
*   **Strengths:** Simplicity, speed, independence from geometry complexity [S1, unofficial].
*   **Limitations:** Quality depends on buffer resolution and precision; may produce jagged edges; does not provide analytical curve data for stylization without additional processing [S1, unofficial].
*   **When to use:** Real-time NPR, simple outline effects, when performance is critical [S1, S2, unofficial].
*   **When not to use:** When depth/normal buffers are unavailable or unreliable [G].

## Background
The method was surveyed by Hertzmann [Her99] and is based on the observation that depth and normal discontinuities correspond to perceptual edges [S1, unofficial]. It is widely used in post-processing pipelines for NPR [S2, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] lazergenixdev/First-Voxel-Engine, https://github.com/lazergenixdev/First-Voxel-Engine, fetched 2026-10-26 (unofficial)
