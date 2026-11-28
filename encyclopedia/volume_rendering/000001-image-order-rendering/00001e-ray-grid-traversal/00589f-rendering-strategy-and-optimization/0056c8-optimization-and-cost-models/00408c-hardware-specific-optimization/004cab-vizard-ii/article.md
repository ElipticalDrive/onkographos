# VIZARD II

Also called: UVIZARD, VIZARD

## Overview
VIZARD II is a specialized hardware architecture for interactive volume rendering, developed at the University of Tübingen to bring ray casting to desktop computers [S2, unofficial]. It solves the memory bandwidth bottleneck inherent in image-order ray casting by combining early ray termination, 9-bit quantized gradient look-up tables, and a specialized caching mechanism [S4]. Unlike its predecessor VIZARD, which used pre-shaded and pre-classified compressed data, VIZARD II computes gradients on-the-fly via a quantized table [S4]. Unlike the VolumePro, it supports perspective rendering while using a table-based gradient scheme instead of hardware gradient estimation [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (voxel values) and a precomputed quantized gradient table [S4].
*   **Output:** A 2D image rendered via perspective or parallel projection, supporting multiple cut-planes and segmentation [S4].

**Data Structures:**
1.  **Volume Data:** Stored in memory banks. Voxels are four-way interleaved with respect to the YZ-plane to facilitate parallel access [S4].
2.  **Gradient Table:** A look-up table mapping 9-bit gradient indices to 3D gradient vectors (x, y, z components) [S4]. The index size is not strictly limited to 9 bits, but 9 bits is the standard implementation [S4].
3.  **Cache Banks:** Separate cache banks store recently accessed voxel neighborhoods and gradient components to allow parallel re-use [S4].

**Algorithm Steps:**
1.  **Preprocessing:** Compute gradient indices for each voxel using central differences. Store these indices in the volume data or a separate structure [S4]. This step is performed once per dataset [S4].
2.  **Ray Initialization:** For each pixel, initialize the ray origin and direction [G].
3.  **Memory Access (Trilinear Neighborhood):**
    *   To sample a voxel, the architecture requires a $2 \times 2 \times 2$ trilinear interpolation neighborhood [S4].
    *   Four voxels (including their gradient indices) are accessed simultaneously in parallel from the memory banks. These voxels correspond to a slice in the YZ-plane [S4].
    *   A sequential burst memory access fetches the adjacent voxels in the X-direction from each bank, providing the remaining four values of the $2 \times 2 \times 2$ neighborhood [S4].
    *   These values are cached in separate cache banks to allow parallel access and re-use [S4].
4.  **Interpolation:**
    *   The **Interpolation Unit** uses the fractional x, y, and z components of the ray position and the cached voxel values to perform trilinear interpolation, yielding the sample value [S4].
    *   The gradient index from the accessed voxels addresses the **Gradient Look-up Table**. The resulting x, y, and z gradient components are interpolated in a similar manner to the voxel values [S4].
    *   The quantized central difference gradient ensures that the voxel re-fetch does not exceed the size of the trilinear interpolation neighborhood [S4].
5.  **Shading and Compositing:**
    *   The sample value and interpolated gradient are passed to the **Shading and Compositing Unit** [S4].
    *   **Classification:** A classification index (derived from the sample value) selects a classification table. The architecture supports segmentation and multiple cut-planes [S4].
    *   **Shading:** Phong shading is implemented using a look-up table based on the gradient and view direction [S4].
    *   **Compositing:** The standard 'over' operator is used to composite the sample into the accumulating ray buffer [S4].
6.  **Early Ray Termination:** If the accumulated opacity exceeds a threshold, the ray is terminated early to increase frame rate [S4].
7.  **Output:** Final pixel values are transferred through the PCI bus to the host computer [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vizard_render(volume, grad_table, ray_origins, ray_dirs, img_h, img_w, max_t, step=0.1, thresh=0.99):
    """
    Simulates VIZARD II hardware pipeline.
    volume: 3D array (D, H, W) of voxel values.
    grad_table: 2D array (256, 3) mapping 9-bit indices to (gx, gy, gz).
    ray_origins, ray_dirs: lists of (x, y, z) tuples.
    """
    img = [[(0.0, 0.0, 0.0, 0.0) for _ in range(img_w)] for _ in range(img_h)]
    
    for py in range(img_h):
        for px in range(img_w):
            ox, oy, oz = ray_origins[py * img_w + px]
            dx, dy, dz = ray_dirs[py * img_w + px]
            
            acc_r, acc_g, acc_b, acc_a = 0.0, 0.0, 0.0, 0.0
            t = 0.0
            
            while t < max_t:
                x, y, z = ox + t * dx, oy + t * dy, oz + t * dz
                
                # Boundary check
                if not (0.0 <= x < W and 0.0 <= y < H and 0.0 <= z < D):
                    break
                
                ix, iy, iz = int(x), int(y), int(z)
                fx, fy, fz = x - ix, y - iy, z - iz
                
                # Hardware Memory Access: 4 parallel YZ-plane accesses + burst X
                # Fetch 8 voxels and 8 gradient indices for 2x2x2 neighborhood
                voxels = []
                grad_idxs = []
                for dz in range(2):
                    for dy in range(2):
                        for dx in range(2):
                            voxels.append(volume[min(iz+dz, D-1)][min(iy+dy, H-1)][min(ix+dx, W-1)])
                            grad_idxs.append(volume_grad_idx[min(iz+dz, D-1)][min(iy+dy, H-1)][min(ix+dx, W-1)])
                
                # Hardware Interpolation Unit: Trilinear interpolation
                sample_val = (
                    voxels[0]*(1-fx)*(1-fy)*(1-fz) + voxels[1]*fx*(1-fy)*(1-fz) +
                    voxels[2]*(1-fx)*fy*(1-fz) + voxels[3]*fx*fy*(1-fz) +
                    voxels[4]*(1-fx)*(1-fy)*fz + voxels[5]*fx*(1-fy)*fz +
                    voxels[6]*(1-fx)*fy*fz + voxels[7]*fx*fy*fz
                )
                
                # Interpolate gradient components from LUT
                grad_vals = [grad_table[gi] for gi in grad_idxs]
                gx = sum(v[0] * (1-fx)*(1-fy)*(1-fz) if i==0 else fx*(1-fy)*(1-fz) if i%2==1 else (1-fx)*fy*(1-fz) if i%4==2 else fx*fy*(1-fz) if i%4==3 else (1-fx)*(1-fy)*fz if i%8==4 else fx*(1-fy)*fz if i%8==5 else (1-fx)*fy*fz if i%8==6 else fx*fy*fz for i, v in enumerate(grad_vals))
                gy = sum(v[1] * (1-fx)*(1-fy)*(1-fz) if i==0 else fx*(1-fy)*(1-fz) if i%2==1 else (1-fx)*fy*(1-fz) if i%4==2 else fx*fy*(1-fz) if i%4==3 else (1-fx)*(1-fy)*fz if i%8==4 else fx*(1-fy)*fz if i%8==5 else (1-fx)*fy*fz if i%8==6 else fx*fy*fz for i, v in enumerate(grad_vals))
                gz = sum(v[2] * (1-fx)*(1-fy)*(1-fz) if i==0 else fx*(1-fy)*(1-fz) if i%2==1 else (1-fx)*fy*(1-fz) if i%4==2 else fx*fy*(1-fz) if i%4==3 else (1-fx)*(1-fy)*fz if i%8==4 else fx*(1-fy)*fz if i%8==5 else (1-fx)*fy*fz if i%8==6 else fx*fy*fz for i, v in enumerate(grad_vals))
                
                # Hardware Shading/Compositing Unit
                class_idx = int(sample_val * 255) % 256
                alpha = sample_val * 0.5
                r, g, b = sample_val, sample_val, sample_val
                
                # 'over' compositing
                if alpha > 0.0:
                    acc_r += (1.0 - acc_a) * r * alpha
                    acc_g += (1.0 - acc_a) * g * alpha
                    acc_b += (1.0 - acc_a) * b * alpha
                    acc_a += (1.0 - acc_a) * alpha
                
                # Early Ray Termination
                if acc_a >= thresh:
                    break
                
                # Advance ray
                t += step
            
            img[py][px] = (acc_r, acc_g, acc_b, acc_a)
    return img
```

**Parameters:**
*   **Gradient Quantization:** 9-bit indices are standard. Larger tables increase accuracy but consume more memory [S4].
*   **Supersampling:** 4:1 mapping of samples to voxels is used in best-case performance estimates [S4].
*   **Clock Rates:** Cache memory and Trilinear Interpolation Units are clocked 4 times faster than the memory system [S4].

**Edge Cases and Pitfalls:**
*   **Worst-Case Performance:** Occurs with 1:1 sampling (no supersampling) and a transparent classification of the dataset, leading to full ray traversal [S4].
*   **Gradient Accuracy:** Quantized central difference gradients may degrade image quality compared to traditional central difference gradients [S4].
*   **Memory Bandwidth:** Performance is memory-limited. The architecture relies on cache hits to sustain high throughput [S4].

## Operating characteristics
*   **Time Complexity:** The algorithmic speedup is assumed to be 3x due to early ray termination [S4].
*   **Memory Efficiency:** 
    *   Worst-case (1:1 sampling): Memory efficiency factor is 0.2 [S4].
    *   Best-case (4:1 supersampling): Memory bandwidth utilization reaches 0.8 due to cache hits [S4].
*   **Performance Metrics:**
    *   **Worst-case:** ~14 million samples per second (SPPS) using four MHz SDRAM devices. This results in approximately 1 frame per second for a 256^3 dataset [S4].
    *   **Best-case:** ~56 million SPPS assuming 4:1 supersampling. This results in approximately 10 Hz frame rate for a 256^3 dataset [S4].
*   **Strengths:** Supports perspective rendering, multiple cut-planes, and segmentation. High performance cost ratio [S4].
*   **Limitations:** Memory bandwidth limited. Quantized gradients may reduce image quality. Research-stage architecture [S4].

## Background
VIZARD II is the second generation of the VIZARD systems, developed at the University of Tübingen [S2, unofficial]. It was designed to bring interactive ray casting to desktop computers using a PCI board populated with four FPGAs, a DSP, and SDRAM/SRAM memory [S3, unofficial]. It is a research-stage architecture [S4]. It differs from the VolumePro by supporting perspective rendering and using a table-based gradient vector lookup scheme [S3, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27 (unofficial)
[S3] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S4] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27
