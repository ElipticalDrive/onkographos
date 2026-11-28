# Adaptive Image Sampling with Pixel Sharing

Also called: Adaptive Image Sampling, Adaptive image sampling optimization, adaptive sampling with synchronization

## Overview
Adaptive Image Sampling with Pixel Sharing is a parallel volume rendering optimization that reduces computational cost by casting rays only from the corners of square image regions and interpolating the interior pixels [S1][S2, unofficial]. It solves the problem of redundant ray casting in smooth areas while managing the synchronization dependencies inherent in parallel implementations where adjacent regions share corner pixels [S1][S2, unofficial]. Unlike non-ray-dependent optimizations such as early ray termination, this method requires careful handling of shared pixel values to avoid replicated computation across processor boundaries [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (voxels), an initial maximum sample region size $w_{max}$, a color difference threshold $T_c$, and a density difference threshold $T_d$ [S1][S2, unofficial].
*   **Output:** A rendered 2D image where pixels are either explicitly ray-traced or bilinearly interpolated from traced corner pixels [S1].

**Data Structures:**
*   **Image Plane Partitioning:** The image is divided into square sample regions of size $w_{max} \times w_{max}$ pixels [S1][S2, unofficial].
*   **Task Queue:** A queue of image tiles (or sample regions) to be processed by processors in a MIMD architecture [S1].
*   **Wait Queue:** A local data structure storing the state of sample regions that cannot be fully evaluated because their corner pixels are currently being traced by other processors [S1].
*   **Preprocessing Data:** Normals, opacities, and an octree structure are computed as preprocessing steps [S1]. A lookup table mapping local voxel gradient to color is used to accelerate shading [S1].

**Algorithm Steps:**
The method operates in two phases: initial processing and wait-queue resolution [S1].

1.  **Region Division:** Divide the image plane into square sample regions of size $w_{max} \times w_{max}$ [S1][S2, unofficial].
2.  **Corner Ray Casting:** For each sample region, identify the four corner pixels [S1]. Cast rays only for those corner pixels that have not already been traced by adjacent regions and are not currently being traced by other processors [S1].
3.  **Complexity Measurement:** Compute the image complexity $M$ of the sample region by calculating the color difference between the successfully traced corner pixels [S1].
    *   Formula: $M = \max(|C_i - C_j|)$ for all pairs of traced corners $i, j$ in the region, where $C$ is the color vector [S1].
    *   Note: Pixels currently being traced by other processors are excluded from this calculation [S1].
4.  **Adaptation Decision:**
    *   If $M > T_c$ (color threshold), subdivide the sample region into smaller subregions and recursively apply the algorithm to them [S1].
    *   If $M \le T_c$, the region is considered smooth [S1].
5.  **Handling Dependencies (Wait Queue):**
    *   If the region requires subdivision but some corner pixels are still being traced by other processors, or if the region does not require subdivision but some corners are still being traced, store the sample region's state in a local **wait queue** [S1].
    *   If all necessary corner pixels are available and the region does not require subdivision, proceed to interpolation (Step 6) [S1].
6.  **Interpolation:** For regions that do not require subdivision, bilinearly interpolate the untraced interior pixels from the traced corner pixels [S1].
7.  **Wait Queue Resolution:**
    *   After all initial task queues are empty, process the local wait queues [S1].
    *   For each sample region in the wait queue, re-evaluate complexity (Step 3) [S1].
    *   If corner pixels are still being traced, **busy wait** until they are ray traced [S1].
    *   Once all corners are available, re-evaluate the complexity and proceed to subdivision or interpolation as determined in Step 4 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_image_sampling(image_width, image_height, w_max, T_c, T_d, volume_data):
    image = [[None] * image_width for _ in range(image_height)]
    task_queue = []
    wait_queue = []

    class SampleRegion:
        def __init__(self, x, y, size):
            self.x, self.y, self.size = x, y, size
        def corners(self):
            s = self.size
            return [(self.x, self.y), (self.x+s, self.y), (self.x, self.y+s), (self.x+s, self.y+s)]

    for y in range(0, image_height, w_max):
        for x in range(0, image_width, w_max):
            task_queue.append(SampleRegion(x, y, w_max))

    def is_traced(px, py): return image[py][px] is not None
    def is_being_traced_by_other(px, py): return False # Parallel state check [G]
    def ray_trace(px, py, vol): return [0.0, 0.0, 0.0] # Actual volume march [G]
    def color_diff(c1, c2): return max(abs(c1[i]-c2[i]) for i in range(3))

    while task_queue:
        reg = task_queue.pop()
        corners = reg.corners()
        traced = []
        for px, py in corners:
            if not is_traced(px, py) and not is_being_traced_by_other(px, py):
                c = ray_trace(px, py, volume_data)
                image[py][px] = c
                traced.append((px, py, c))

        if not traced:
            wait_queue.append(reg)
            continue

        max_d = 0
        for i in range(len(traced)):
            for j in range(i+1, len(traced)):
                d = color_diff(traced[i][2], traced[j][2])
                if d > max_d: max_d = d

        if max_d > T_c:
            s = reg.size
            if s > 1:
                h = s // 2
                task_queue.append(SampleRegion(reg.x, reg.y, h))
                task_queue.append(SampleRegion(reg.x+h, reg.y, h))
                task_queue.append(SampleRegion(reg.x, reg.y+h, h))
                task_queue.append(SampleRegion(reg.x+h, reg.y+h, h))
        else:
            if len(traced) < 4:
                wait_queue.append(reg)
            else:
                c00, c10, c01, c11 = traced[0][2], traced[1][2], traced[2][2], traced[3][2]
                s = reg.size
                for py in range(reg.y, reg.y+s):
                    for px in range(reg.x, reg.x+s):
                        u, v = (px-reg.x)/s, (py-reg.y)/s
                        top = [c00[k]*(1-u)+c10[k]*u for k in range(3)]
                        bot = [c01[k]*(1-u)+c11[k]*u for k in range(3)]
                        image[py][px] = [top[k]*(1-v)+bot[k]*v for k in range(3)]

    while wait_queue:
        reg = wait_queue.pop()
        corners = reg.corners()
        while any(not is_traced(px, py) for px, py in corners):
            pass # Busy wait [S1]
        traced = [(px, py, image[py][px]) for px, py in corners]
        max_d = 0
        for i in range(len(traced)):
            for j in range(i+1, len(traced)):
                d = color_diff(traced[i][2], traced[j][2])
                if d > max_d: max_d = d
        if max_d > T_c:
            s = reg.size
            if s > 1:
                h = s // 2
                task_queue.append(SampleRegion(reg.x, reg.y, h))
                task_queue.append(SampleRegion(reg.x+h, reg.y, h))
                task_queue.append(SampleRegion(reg.x, reg.y+h, h))
                task_queue.append(SampleRegion(reg.x+h, reg.y+h, h))
        else:
            c00, c10, c01, c11 = traced[0][2], traced[1][2], traced[2][2], traced[3][2]
            s = reg.size
            for py in range(reg.y, reg.y+s):
                for px in range(reg.x, reg.x+s):
                    u, v = (px-reg.x)/s, (py-reg.y)/s
                    top = [c00[k]*(1-u)+c10[k]*u for k in range(3)]
                    bot = [c01[k]*(1-u)+c11[k]*u for k in range(3)]
                    image[py][px] = [top[k]*(1-v)+bot[k]*v for k in range(3)]
    return image
```

**Parameters:**
*   $w_{max}$: The side length in pixels of the initial square sample regions. Implementations use $3 \times 3$ pixels [S2, unofficial].
*   $T_c$: Color difference threshold for subdivision. Typical values: 10 (electric field), 40 (neghip), 30 (brain) [S2, unofficial].
*   $T_d$: Density difference threshold for sample point interpolation along rays. Typical values: 30 (electric field), 80 (neghip), 5 (brain) [S2, unofficial].
*   Early Ray Termination (ERT) Opacity Threshold: Often used in conjunction. Typical values: 98% (electric field), 92% (neghip), 98% (brain) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Pixel Sharing Dependencies:** Adjacent sample regions share corner pixels [S1][S2, unofficial]. If processed by different processors, synchronization is required to avoid replicated ray casting [S1][S2, unofficial].
*   **Deadlock/Starvation:** If a processor waits indefinitely for a pixel traced by another, busy waiting or proper queue management is required [S1].
*   **Precision:** Color difference calculations must be robust to small numerical variations [S1].

## Subtopics

### [Static Contiguous Image-Based Partitioning](11259-static-contiguous-image-based-partitioning.md)
Static Contiguous Image-Based Partitioning is a parallel volume rendering strategy that divides the final image plane into contiguous blocks (tiles) and assigns each block to a specific processor, which then executes the complete rendering pipeline for that region. It solves the load-balancing and communication overhead issues inherent in adaptive sampling methods by ensuring that pixel-sharing costs are eliminated through static assignment and specific traversal orders. Unlike dynamic image-based partitioning which delays evaluation to handle shared pixels, this method avoids such costs entirely without extra synchronization overhead.

## Operating characteristics
*   **Time Complexity:** Reduces the number of ray casts in smooth regions by interpolating interior pixels, leading to faster rendering times [S1][S2, unofficial].
*   **Memory Complexity:** Requires storage for the image buffer, task queues, wait queues, and preprocessing data (octree, normals, opacities) [S1].
*   **Measured Performance:** On a 16-processor system, rendering rates of >10 frames per second were achieved for $256 \times 256$ resolution images on datasets like electric field, neghip, and brain [S2, unofficial].
*   **Strengths:** Significant reduction in computational cost for smooth regions; avoids replicated ray casting in parallel environments through pixel sharing [S1][S2, unofficial].
*   **Limitations:** Requires synchronization or queue management for parallel implementations; interpolation artifacts may occur in regions with high complexity if thresholds are not tuned correctly [S1][S2, unofficial].
*   **When to Use:** Suitable for parallel volume rendering on MIMD architectures where smooth regions dominate the image [S1][S2, unofficial].
*   **When Not to Use:** Not suitable for highly complex, noisy, or high-frequency regions where interpolation would introduce significant artifacts [S1].

## Background
The method was introduced in the context of volume rendering on scalable shared-memory MIMD architectures, specifically the DASH multiprocessor [S1]. It addresses the inefficiencies of uniform sampling by adapting the sampling rate based on image complexity [S1]. The approach was further optimized for parallel implementations by handling pixel sharing dependencies, leading to static contiguous image-based partitioning strategies [S2, unofficial].

## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
[S2] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
