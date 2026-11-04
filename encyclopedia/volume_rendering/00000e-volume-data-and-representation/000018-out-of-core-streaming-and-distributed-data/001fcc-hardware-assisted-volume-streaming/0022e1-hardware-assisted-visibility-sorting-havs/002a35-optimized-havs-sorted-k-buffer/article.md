# Optimized HAVS (Sorted k-buffer)

Also called: Optimization of HAVS, Sorted k-buffer HAVS

## Overview
Optimized HAVS is a refinement of Hardware Assisted Visibility Sorting [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md) that maintains the k-buffer in a strictly sorted state (ascending order of fragment distance) to reduce the computational cost of fragment-level visibility sorting. While standard HAVS requires $2k-1$ comparisons to identify the two nearest fragments for compositing, this optimization reduces that requirement to $k$ comparisons per fragment processing step after the buffer is full, thereby improving rendering efficiency for large unstructured grid datasets.

## How it works
**Inputs:**
- A stream of volume fragments projected to screen space.
- A k-buffer of fixed capacity $k$ per pixel, storing the nearest fragments.
- A 3D pre-integrated lookup table for color and opacity retrieval.
- Image dimensions $m \times n$.

**Outputs:**
- A composited color and opacity value for each pixel in the framebuffer.
- The k-buffer state updated for the next frame or fragment batch.

**Data Structures:**
- **k-buffer:** An array of size $k$ per pixel. Each element stores fragment properties (distance, color, opacity, etc.). The buffer is maintained in ascending order of distance from the camera.
- **3D Pre-integrated Table:** A texture or lookup table mapping distance intervals and optical properties to color/opacity values.

**Parameters:**
- $k$: The number of nearest fragments to retain. The source notes that existing HAVS restricts $k$ to 6, but this optimization allows for analysis of $k$ ranges to balance precision and efficiency. Typical values are small integers (e.g., $k \le 6$ or slightly higher depending on implementation constraints).

**Algorithm Steps:**
For each pixel $j$ in the image ($1 \le j \le m \times n$), let $s_j$ be the total number of fragments projected to that pixel. The k-buffer is processed as follows:

1. **Buffer Not Full ($current\_size < k$):**
   - Insert the new fragment into the k-buffer at the correct position to maintain ascending order of distance.
   - This requires comparing the new fragment with existing fragments to find the insertion point.

2. **Buffer Full ($current\_size == k$):**
   - Let the fragments in the buffer be $f_1, f_2, \dots, f_k$, where $f_1$ is the nearest (smallest distance) and $f_k$ is the farthest.
   - Select $f_1$ as the first nearest fragment candidate.
   - Select the nearest fragment $f_i$ from the remaining $k-1$ fragments ($f_2, \dots, f_k$). Since the buffer is sorted, $f_i$ is simply $f_2$.
   - Compare $f_1$ and $f_i$ (i.e., $f_1$ and $f_2$). The smaller one (nearest) is $f_j$.
   - Retrieve color and opacity using the pair $(v_i, v_1)$ (representing the volume intervals) and the distance difference from the 3D pre-integrated table.
   - Composite the resulting color and opacity to the framebuffer.
   - Discard $f_j$ (the nearest fragment, which has been composited) and write the remaining $k-1$ fragments back to the k-buffer.
   - Insert the new incoming fragment into the sorted buffer.

**Formulas:**
Let $N_2$ be the total number of comparisons for the sorted k-buffer across all pixels.
Let $t_1$ be comparisons during insertion (buffer not full).
Let $t_2$ be comparisons when the buffer is full.

The total comparisons $N_2$ are given by:
$$ N_2 = \sum_{j=1}^{m \times n} (t_1 + t_2) = \sum_{j=1}^{m \times n} \left( \sum_{l=1}^{k-1} l + (1 + k - 1) \times (s_j - k) \right) $$

Where:
- $\sum_{l=1}^{k-1} l$ represents the comparisons needed to insert fragments into the buffer while it is filling up (sum of $1$ to $k-1$).
- $(1 + k - 1) \times (s_j - k)$ represents the comparisons for the remaining $s_j - k$ fragments after the buffer is full. The term $(1 + k - 1)$ simplifies to $k$, which is the number of comparisons per fragment when the buffer is full.

The average comparisons per pixel $e_2$ are:
$$ e_2 = \frac{N_2}{m \times n} = \frac{\sum_{j=1}^{m \times n} (\sum_{l=1}^{k-1} l + k \times (s_j - k))}{m \times n} $$

The difference in average comparisons between the original HAVS ($e_1$) and the optimized HAVS ($e_2$) is:
$$ \Delta e = e_1 - e_2 = \frac{\sum_{j=1}^{m \times n} ((k-1) \times (s_j - \frac{k}{2} - 1))}{m \times n} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimized_havs_fragments(fragments, k, pre_integrated_table, image_width, image_height):
    """
    Process fragments using Optimized HAVS with a sorted k-buffer.
    
    Args:
        fragments: List of fragments, each with (distance, color, opacity, etc.)
        k: Buffer size
        pre_integrated_table: Function/lookup for color/opacity from distance intervals
        image_width: m
        image_height: n
        
    Returns:
        framebuffer: Array of composited colors
    """
    # Initialize k-buffers for each pixel
    # k_buffer[pixel_index] is a list of fragments, sorted by distance
    k_buffers = [[] for _ in range(image_width * image_height)]
    framebuffer = [None] * (image_width * image_height)
    
    # Group fragments by pixel
    pixel_fragments = [[] for _ in range(image_width * image_height)]
    for frag in fragments:
        pixel_idx = frag.pixel_index
        pixel_fragments[pixel_idx].append(frag)
        
    for pixel_idx in range(image_width * image_height):
        frags = pixel_fragments[pixel_idx]
        buffer = k_buffers[pixel_idx]
        
        # Sort fragments by distance for processing order (nearest first)
        # Note: In hardware-assisted streaming, fragments might arrive in order.
        # Assuming we process them in distance order for correctness.
        frags.sort(key=lambda f: f.distance)
        
        for frag in frags:
            current_size = len(buffer)
            
            if current_size < k:
                # Insert into sorted position
                # Find insertion index
                insert_idx = 0
                while insert_idx < current_size and buffer[insert_idx].distance < frag.distance:
                    insert_idx += 1
                buffer.insert(insert_idx, frag)
                
            else:
                # Buffer is full
                # f1 is the nearest (buffer[0])
                # fi is the nearest among the rest (buffer[1])
                f1 = buffer[0]
                fi = buffer[1]
                
                # Compare f1 and fi, get smaller one fj
                # Since buffer is sorted, f1 is always <= fi
                fj = f1
                
                # Use (vi, v1) and distance difference to get color/opacity
                # vi corresponds to fi, v1 corresponds to f1
                # This step assumes pre-integrated table takes volume intervals
                color, opacity = pre_integrated_table(fi.volume_interval, f1.volume_interval, fi.distance - f1.distance)
                
                # Composite to framebuffer
                # Assuming framebuffer stores current accumulated color/opacity
                if framebuffer[pixel_idx] is None:
                    framebuffer[pixel_idx] = (color, opacity)
                else:
                    prev_color, prev_opacity = framebuffer[pixel_idx]
                    # Standard alpha compositing: C_out = C_fg * alpha_fg + C_bg * (1 - alpha_fg)
                    new_color = color * opacity + prev_color * (1 - opacity)
                    new_opacity = opacity + prev_opacity * (1 - opacity)
                    framebuffer[pixel_idx] = (new_color, new_opacity)
                
                # Discard fj (f1) and write remaining k-1 fragments back
                # Remove f1 from buffer
                buffer.pop(0)
                
                # Insert the new fragment into the sorted buffer
                insert_idx = 0
                while insert_idx < k - 1 and buffer[insert_idx].distance < frag.distance:
                    insert_idx += 1
                buffer.insert(insert_idx, frag)
                
    return framebuffer
```

## Operating characteristics
- **Time Complexity:** The number of comparisons per fragment after the buffer is full is $k$, compared to $2k-1$ in the original HAVS. The total comparisons $N_2$ are reduced, leading to improved efficiency. The improvement ratio $\frac{\Delta e}{e_1}$ increases as the dataset size ($s_j$) increases.
- **Memory Complexity:** Requires storing $k$ fragments per pixel in the k-buffer.
- **Strengths:** Reduces comparison overhead, especially for large datasets with many fragments per pixel. Allows for larger $k$ values without proportional performance loss, potentially improving rendering precision.
- **Limitations:** Requires maintaining the sorted order of the k-buffer, which involves insertion operations. The benefit is most pronounced when $s_j$ is significantly larger than $k$.

## Background
This optimization was proposed in a 2012 paper by authors associated with ICCASM 2012 [S1]. It addresses the inefficiency of the original HAVS algorithm, which requires $2k-1$ comparisons to find the two nearest fragments. By keeping the k-buffer sorted, the algorithm reduces this to $k$ comparisons. The paper also investigates the optimal range for $k$ to balance precision and efficiency, noting that standard HAVS often restricts $k$ to 6.

## Sources
[S1] Optimization and Implementation of Unstructured Grid Volume Rendering Algorithm, https://doi.org/10.2991/iccasm.2012.373, fetched 2026-10-26
