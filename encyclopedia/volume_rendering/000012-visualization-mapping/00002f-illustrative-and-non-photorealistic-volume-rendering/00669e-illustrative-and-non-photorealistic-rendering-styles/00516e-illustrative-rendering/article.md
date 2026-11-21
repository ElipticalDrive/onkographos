# Illustrative Rendering

Also called: Illustrative visualization, Streamtape illustrative rendering

## Overview
Illustrative rendering is a non-photorealistic visualization technique that combines Phong shading, half-tone hatching shadows, and silhouette outlines to enhance the depth and structural perception of volumetric objects, specifically streamtapes in 3D vector fields [S1]. It differs from procedural texture-based methods (e.g., Hummel et al.) by avoiding complex line patterns on narrow objects, which would increase visual complexity, instead relying on stylized lighting and edge detection to clarify geometry [S1].

## How it works
**Inputs:**
- 3D vector field data.
- Pre-computed streamtapes (narrow tubular surfaces representing flow).
- Camera position and light direction.

**Outputs:**
- Final illustrative image combining Phong shading, hatched shadows, and silhouette outlines.

**Data Structures:**
- Framebuffer Object (FBO) for storing luminance.
- Color buffer for storing normalized normal vectors (RGB) and clamped depth (Alpha).
- Discontinuity mask buffer for silhouette detection.

**Parameters:**
- Torsion normalization range: $[0, 1]$ [S1].
- Streamtape width formula: $\omega = 1.0 - \tau$, where $\tau$ is normalized torsion [S1].
- Discontinuity threshold: Not given in the sources.

**Steps:**

1. **Phong Shading Pass:** Render the streamtapes using standard Phong shading. Record the resulting luminance into a framebuffer object [S1].

2. **Hatched Shadow Generation:** Generate a hatched shadow image using half-tone rendering styles to incorporate lighting and display depth/rotation information [S1]. This step replaces standard diffuse shading with a stylized hatch pattern [S1].

3. **Silhouette Extraction Pass:** Render the scene again to obtain silhouettes. In this pass:
   - Write the normalized normal direction of each fragment to the RGB channels of the output color.
   - Clamp the depth value to $[0, 1]$ and write it to the alpha channel [S1].

4. **Silhouette Detection:** Check for discontinuities in the up, down, left, and right directions in image space on the output from Step 3. Mark pixels with large discontinuities as silhouette pixels [S1].

5. **Composition:** Compose the hatched shadow image and the silhouette image with the Phong shaded image [S1].

6. **Coloring:** Color the streamtapes either by cluster ID using a predefined color table or by local features such as curvature [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def illustrative_render(streamtapes, camera, light_dir):
    # Step 1: Phong Shading
    phong_luminance = render_phong(streamtapes, camera, light_dir)
    
    # Step 2: Hatched Shadow
    # Uses half-tone rendering styles to incorporate lighting and depth/rotation
    hatched_shadow = render_hatched_shadow(streamtapes, camera, light_dir)
    
    # Step 3: Silhouette Extraction Pass
    # Render normals and clamped depth
    normal_depth_buffer = render_normals_and_depth(streamtapes, camera)
    
    # Step 4: Silhouette Detection
    # Check discontinuities in 4 directions (up, down, left, right)
    silhouette_mask = detect_discontinuities(normal_depth_buffer)
    
    # Step 5: Composition
    # Combine Phong, hatched shadow, and silhouette outlines
    final_image = compose_images(phong_luminance, hatched_shadow, silhouette_mask)
    
    # Step 6: Coloring
    # Option A: Cluster ID
    # Option B: Local features (e.g., curvature)
    final_image = apply_coloring(final_image, streamtapes)
    
    return final_image

def render_normals_and_depth(streamtapes, camera):
    buffer = empty_buffer()
    for fragment in streamtapes:
        normal = normalize(fragment.normal)
        depth = clamp(fragment.depth, 0.0, 1.0)
        buffer.rgb = normal
        buffer.a = depth
    return buffer

def detect_discontinuities(buffer):
    mask = empty_buffer()
    for pixel in buffer:
        # Check neighbors in up, down, left, right directions
        if has_large_discontinuity(pixel, buffer):
            mask.mark_silhouette(pixel)
    return mask
```

**Edge Cases and Pitfalls:**
- **Self-Intersection:** When local flow twists sharply, streamtape surfaces can self-intersect. This is addressed by normalizing torsion $\tau$ to $[0, 1]$ and setting width $\omega = 1.0 - \tau$ [S1].
- **Visual Complexity:** Procedural line stripes (e.g., Hummel et al.) are incompatible because streamtapes are narrow objects where complex patterns increase visual complexity [S1].

## Operating characteristics
- **Time Complexity:** Entropy field calculation and streamline generation are implemented on the GPU (CUDA) and complete within a few seconds for given flow datasets [S1]. K-means clustering and streamtape generation are implemented on the CPU [S1].
- **Memory:** Requires 1.5GB of video memory for the GPU (tested on Geforce GTX 480) and 16GB system memory (tested on Xeon 5450) [S1].
- **Strengths:** Enhances depth and structural perception; reveals structural information via curvature-based coloring [S1].
- **Limitations:** Not suitable for integral surfaces requiring adaptive line stripes; limited to narrow objects like streamtapes [S1].

## Background
The method was introduced in 2011 as part of an illustrative visualization framework for 3D vector fields [S1]. It was implemented on the GPU and used to visualize streamtapes generated from vector field data [S1]. The half-tone rendering style references the work of Freudenberg et al. [FMS01] for an overview of half-tone rendering techniques [S1].

## Sources
[S1] An Illustrative Visualization Framework for 3D Vector Fields, http://vis.cs.ucdavis.edu/papers/pg2011paper.pdf, fetched 2026-10-29
