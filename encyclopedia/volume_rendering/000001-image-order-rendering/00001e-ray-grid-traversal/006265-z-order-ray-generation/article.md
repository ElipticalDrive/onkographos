# Z-order Ray Generation

Also called: Z-order Ray Scanning, Z-order curve ray generation, ZORG (Z-order Ray Generation)

## Overview
Z-order Ray Generation (ZORG) is a hardware-oriented ray generation method that produces pixel coordinates and ray directions by traversing the image plane along a Z-order (Morton) curve rather than in row-major order [S1]. It clusters four adjacent rays into Ray Packets (RPs) to enhance spatial locality, thereby increasing traversal path similarity and data access overlap for occupancy grids and feature vectors compared to row-order scanning [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An external system provides the image dimensions and the current state of the Z-order counter.
*   **Output:** Four sets of pixel coordinates $(X, Y)$ and their corresponding ray directions (origin and direction vectors) forming a single Ray Packet (RP) [S1].

**Data Structures:**
*   **Z-order Counter:** A 22-bit unsigned integer counter [S1].
*   **Pixel Coordinates:** Two 11-bit binary values, $CX$ (X-coordinate) and $CY$ (Y-coordinate), derived from the counter [S1].

**Steps:**
1.  **Increment Counter:** The 22-bit counter is incremented to generate the next value in the Z-order sequence [S1].
2.  **Bit Interleaving (De-interleaving):** The counter output, denoted as $FC$, is partitioned into odd and even bits to form the X and Y coordinates [S1]:
    *   $CX$ is formed by the even bits of $FC$ (bits 0, 2, 4, ..., 20).
    *   $CY$ is formed by the odd bits of $FC$ (bits 1, 3, 5, ..., 21).
    *   This effectively reverses the Morton encoding process to retrieve spatial coordinates from the linear Z-order index [S1].
3.  **Ray Direction Computation:** Using the generated $(CX, CY)$ coordinates, the ray origin and direction are computed for each of the four rays in the packet [S1]. The specific geometric mapping depends on the camera model (e.g., pinhole), but the coordinates define the pixel centers or corners [S1].
4.  **Boundary Handling:** If the generated coordinates fall outside the valid image boundaries (e.g., for non-power-of-two resolutions like 800x800), the system detects out-of-bounds positions and preemptively corrects the coordinates to re-enter the valid region [S1].
5.  **Packet Assembly:** The four rays are assembled into an RP and passed to the AABB Tester (ABT) for culling [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zorder_ray_generation(counter_width=22, packet_size=4):
    """
    Generates ray coordinates for a packet of rays using Z-order traversal.
    Note: This is a simplified logical representation. The actual hardware
    implementation uses bit-parallel de-interleaving.
    """
    # Initialize counter
    counter = 0
    
    # Precompute valid bounds if known, otherwise assume power-of-two for simplicity
    # In practice, bounds checking is integrated into the coordinate generation
    valid_x_max = None 
    valid_y_max = None
    
    rays = []
    
    for _ in range(packet_size):
        # 1. Get current Z-order index
        z_index = counter
        
        # 2. De-interleave bits to get X and Y
        # Extract even bits for X
        cx = 0
        bit_pos = 0
        temp_z = z_index
        while temp_z > 0:
            if temp_z & 1:
                cx |= (1 << bit_pos)
            temp_z >>= 2
            bit_pos += 1
            
        # Extract odd bits for Y
        cy = 0
        bit_pos = 0
        temp_z = z_index >> 1
        while temp_z > 0:
            if temp_z & 1:
                cy |= (1 << bit_pos)
            temp_z >>= 2
            bit_pos += 1
            
        # 3. Boundary Check and Correction [G]
        # If dimensions are not power-of-two, coordinates may be out of bounds.
        # The hardware analyzes correlation between out-of-bounds positions 
        # and re-entry points to correct coordinates preemptively [S1].
        # Here we assume a simple clamp for logical completeness, 
        # though the source describes a specific correction mechanism.
        if valid_x_max is not None:
            cx = min(cx, valid_x_max)
        if valid_y_max is not None:
            cy = min(cy, valid_y_max)
            
        # 4. Compute Ray Direction [S1]
        # This step depends on the specific camera projection model.
        # For a standard pinhole camera with focal length f and image center (cx0, cy0):
        # direction_x = (cx - cx0) / f
        # direction_y = (cy - cy0) / f
        # direction_z = -1.0
        # origin = (0, 0, 0)
        
        ray = {
            'pixel_x': cx,
            'pixel_y': cy,
            'direction': None # Placeholder for projection math
        }
        rays.append(ray)
        
        # 5. Increment Counter
        counter += 1
        
    return rays
```

**Parameters:**
*   **Counter Width:** 22 bits [S1].
*   **Coordinate Bit Width:** 11 bits per axis [S1].
*   **Packet Size:** 4 rays per Ray Packet (RP) [S1].

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Dimensions:** For image dimensions that are not powers of two (e.g., 800x800), the Z-order curve extends beyond valid image boundaries, producing out-of-bounds coordinates before re-entering the valid region [S1]. The implementation must handle this via preemptive coordinate correction based on the correlation between out-of-bounds positions and re-entry points [S1].

## Operating characteristics
*   **Locality:** Z-order scanning provides greater traversal path similarity and increased data access overlap to the occupancy grid, feature vectors, and MLP weights compared to row-order scanning [S1].
*   **Performance:** Z-order ray scanning reduces Estimated Memory Accesses (EMAs) by 65.4% compared to row-order scanning [S1].
*   **Hardware Efficiency:** The method is designed for microarchitectural implementation, utilizing a simple 22-bit counter and bit-partitioning logic, which is efficient for on-chip data reuse [S1].
*   **Limitations:** The irregular pattern of Z-order traversal may complicate simple sequential memory access patterns if not carefully managed by the scheduler [S1].

## Background
Z-order Ray Generation was introduced as part of an energy-efficient edge coprocessor for neural rendering with explicit data reuse strategies [S1]. It was developed to address the inefficiencies of row-order scanning in ray packet processing, specifically targeting spatial locality optimization to enhance on-chip data reuse [S1]. The method is integrated into a scheduler that clusters rays into Ray Packets (RPs) and processes them using a Hierarchical Ray Marching (HRM) framework [S1].

## Sources
[S1] An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies, https://doi.org/10.1109/tvlsi.2025.3641653, fetched 2026-10-26
