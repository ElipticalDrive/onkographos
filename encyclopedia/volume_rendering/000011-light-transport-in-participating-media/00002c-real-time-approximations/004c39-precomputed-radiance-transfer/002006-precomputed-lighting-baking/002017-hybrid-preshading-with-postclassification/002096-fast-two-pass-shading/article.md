# Fast Two-Pass Shading

Also called: P-buffer shading, Two-pass shading

## Overview
Fast Two-Pass Shading is a rendering optimization that avoids redundant per-fragment computations—specifically volume texture access, classification, and lighting—by sharing intermediate results between pixels covered by the same splat [S1, unofficial][S2]. It decouples the classification and illumination steps from the geometric filtering step by storing intermediate results in a render target (P-buffer) [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A slice of voxels to be rendered as splats.
*   **Output:** A composited image of the slice.
*   **Intermediate Data:** A P-buffer (render target) storing intermediate results for each splat.

**Data Structures:**
The P-buffer stores one entry per splat. Each entry contains the results of volume texture access, voxel classification (via lookup table), and illumination computation [S1, unofficial][S2].

**Numbered Steps:**
1.  **First Pass (P-buffer Generation):** Project each voxel of the current slice as a single-pixel point [S1, unofficial][S2]. For each point splat, compute volume texture access, voxel classification, and illumination [S1, unofficial][S2]. Store these intermediate results in the P-buffer (render target) [S1, unofficial][S2].
2.  **Second Pass (Splat Rendering):** Project the full splat geometry (covering multiple pixels, e.g., 2×2 to 40×40 pixels depending on viewpoint) [S1, unofficial][S2]. For each fragment covered by the splat, perform per-fragment lighting and EWA filtering while reusing the intermediate results from the P-buffer via lookup [S1, unofficial][S2].
3.  **Final Composition:** Composite all splats from the slice into the final image [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_two_pass_shading(slice_of_voxels, view_matrix, splat_radius):
    # Initialize P-buffer render target as a dictionary mapping screen coords to intermediates
    p_buffer = {}

    # Pass 1: Compute intermediate results for each splat
    for voxel in slice_of_voxels:
        # Project voxel center to screen as a single-pixel point
        screen_x, screen_y = project_to_screen(voxel.center, view_matrix)
        key = (int(screen_x), int(screen_y))

        # Compute intermediate results
        texture_sample = sample_volume(voxel.position)
        classification = classify_voxel(texture_sample)
        illumination = compute_illumination(classification, voxel.position)

        # Store in P-buffer
        p_buffer[key] = {
            'texture_sample': texture_sample,
            'classification': classification,
            'illumination': illumination
        }

    # Pass 2: Render full splats with filtering
    final_image = initialize_image()
    for voxel in slice_of_voxels:
        # Project full splat geometry
        splat_center_x, splat_center_y = project_to_screen(voxel.center, view_matrix)
        radius = splat_radius  # Varies from 2 to 40 pixels depending on viewpoint [S1, unofficial][S2]

        # Render fragments covered by the splat
        for dy in range(-radius, radius + 1):
            for dx in range(-radius, radius + 1):
                frag_x, frag_y = splat_center_x + dx, splat_center_y + dy
                if not in_image_bounds(frag_x, frag_y, final_image):
                    continue

                # Lookup intermediate results from P-buffer
                p_key = (int(splat_center_x), int(splat_center_y))
                intermediate = p_buffer.get(p_key)
                if intermediate is None:
                    continue

                # Apply per-fragment lighting and EWA filtering
                weight = compute_ewa_weight(dx, dy, radius)
                final_color = apply_ewa_filtering(intermediate, weight, frag_x, frag_y)

                # Composite into image
                composite_into_image(final_color, frag_x, frag_y, final_image)

    return final_image

def project_to_screen(position, view_matrix):
    # Standard perspective projection
    projected = transform(position, view_matrix)
    return projected.x / projected.w, projected.y / projected.w

def sample_volume(position):
    # Trilinear interpolation from volume texture
    return volume_texture.sample(position)

def classify_voxel(texture_value):
    # Lookup table classification
    return classification_lut[texture_value]

def compute_illumination(classification, position):
    # Standard lighting model using classification and position
    return lighting_model(classification, position)

def compute_ewa_weight(dx, dy, radius):
    # Elliptical Weighted Average weight calculation
    return exp(-(dx*dx + dy*dy) / (2 * radius * radius))

def apply_ewa_filtering(intermediate, weight, frag_x, frag_y):
    # Reuse intermediate results with EWA weight
    return intermediate['illumination'] * weight

def composite_into_image(color, x, y, image):
    # Alpha blending into final image
    image[y][x] = blend(image[y][x], color)
```

**Edge Cases and Pitfalls:**
*   **Context Switches:** The additional rendering pass increases the number of context switches, which can offset some performance gains [S1, unofficial][S2].
*   **Splat Size:** Splat sizes vary from 2×2 to as many as 40×40 pixels depending on the viewpoint [S1, unofficial][S2].

## Operating characteristics
*   **Performance Improvement:** 5%–10% [S1, unofficial][S2].
*   **Hardware:** Measured on a 2.4 GHz P4 system with 2 GB memory and an ATI 9800 Pro video card (256 MB local memory) using DirectX 9.0 SDK [S1, unofficial][S2].
*   **Strengths:** Reduces redundant computation for pixels covered by the same splat [S1, unofficial][S2].
*   **Limitations:** Increased context switches due to the extra pass [S1, unofficial][S2].

## Background
*   **Origin:** Proposed in "Hardware-Accelerated Adaptive EWA Volume Splatting" by Wei Chen et al. [S1, unofficial][S2].
*   **History:** Implemented with DirectX 9.0 SDK under Windows XP [S1, unofficial][S2].
*   **Usage:** Used in volume rendering for scientific visualization, specifically for EWA volume splatting [S1, unofficial][S2].

## Sources
[S1] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
