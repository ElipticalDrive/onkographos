# ATLANTIS FPGA Volume Rendering Architecture

Also called: ATLANTIS computing board, hybrid FPGA/RISC volume rendering

## Overview
The ATLANTIS architecture is a hardware-accelerated volume rendering system that utilizes Field-Programmable Gate Arrays (FPGAs) to implement special-purpose processors for distinct stages of the rendering pipeline [S1]. It solves the computational bottleneck of general-purpose rendering by reconfiguring FPGAs on-the-fly to minimize hardware footprint while maximizing throughput for specific tasks such as distance calculation and ray casting [S1]. Unlike static ASIC implementations, ATLANTIS leverages reconfigurability to handle varying algorithmic requirements, though it is constrained by memory bandwidth limits common to FPGA-based designs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume dataset (stored in external memory/subboard), ray parameters (origin, direction, length), transfer function (LUT), and reflectance map [S1].
*   **Output:** Rendered 2D image (framebuffer) [S1].

**Data Structures and Layout:**
The system relies on a parameter RAM located within the FPGA fabric to store ray-specific data [S1]. For each ray in a multithreading bunch, the following fields are stored [S1]:
*   **Starting Point:** 3 components $\times$ 24 bits/component = 72 bits [S1].
*   **Minimal Distance to Next Sampling Point:** 3 components $\times$ 16 bits/component = 48 bits [S1].
*   **Maximal Ray Length:** 10 bits [S1].
*   **Total per Ray:** 130 bits [S1].

**Configuration Modes:**
The system operates in four distinct configuration modes, each requiring a reconfiguration time between 35 ms and 140 ms depending on the number of FPGAs involved [S1]:
1.  **Initialization:** Loads data into registers and memory [S1].
2.  **LUT and Reflectance Map Adjusting:** Configures transfer functions and shading parameters [S1].
3.  **Distance Calculation:** Reconfigures an FPGA with a two-pass non-linear digital filter to compute distance transforms [S1].
4.  **Rendering Configuration:** Sets internal parameters for the shading process and voxel gray-value-to-opacity assignment [S1].

**Algorithm Steps:**
1.  **Distance Transform Calculation:**
    *   The system determines the extension of transparent regions within the volume dataset using distance transforms [S1].
    *   One FPGA is reconfigured to execute a two-pass non-linear digital filter [S1].
    *   This step optimizes the ray casting by identifying empty space to skip sampling [S1].
2.  **Rendering Configuration and Execution:**
    *   The FPGA is reconfigured into the "Rendering configuration" mode [S1].
    *   This mode executes three tasks in parallel [S1]:
        1.  **Downloading of Ray Parameters:** Transfers ray data from host to FPGA local storage [S1].
        2.  **Implementation of the Rendering Algorithm:** Performs the actual volume ray casting [S1].
        3.  **Upload of Rendering Results:** Sends the final pixel values back to the host [S1].
3.  **Multithreading and Parallelism:**
    *   The system uses a multithreading granularity of an $8 \times 8$ bunch of rays, totaling 64 rays [S1].
    *   To overlap the latency of downloading parameters with the rendering time, an additional local storage (implemented within the FPGA) is utilized [S1].
4.  **Compositing Formula:**
    *   Front-to-back volume compositing is applied per sample $i$ along the ray [G]:
        $$C_{out} = \sum_{i=0}^{N-1} c_i \alpha_i \prod_{j=0}^{i-1} (1 - \alpha_j)$$
        where $c_i$ is the sampled color, $\alpha_i$ is the sampled opacity, and $N$ is the total number of samples traversed [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# ATLANTIS Rendering Pipeline
# Note: This pseudocode represents the logical flow described in [S1].
# Actual hardware implementation involves parallel hardware blocks.

def atlantis_render(volume_data, transfer_function, reflectance_map, distance_map, image_width, image_height, ray_step=0.5):
    # 1. Initialization Phase
    # Reconfigure FPGA for Initialization mode (35-140ms) [S1]
    # Load volume data pointers and basic registers [S1]
    fpga_configure("INITIALIZATION")
    
    # 2. Transfer Function Setup
    # Reconfigure FPGA for LUT/Reflectance mode [S1]
    # Load Transfer Function (LUT) and Reflectance Map into memory [S1]
    fpga_configure("LUT_AND_REFLECTANCE_ADJUSTING")
    
    # 3. Distance Transform Calculation
    # Reconfigure FPGA for Distance Calculation mode [S1]
    # Execute two-pass non-linear digital filter on volume data [S1]
    # Result: Distance map indicating transparent region extensions [S1]
    fpga_configure("DISTANCE_CALCULATION")
    
    # 4. Rendering Loop
    # Reconfigure FPGA for Rendering mode [S1]
    fpga_configure("RENDERING_CONFIGURATION")
    
    # Initialize local storage for ray parameters [S1]
    ray_param_ram = {}
    
    # Process rays in 8x8 bunches (64 rays) [S1]
    for y in range(0, image_height, 8):
        for x in range(0, image_width, 8):
            ray_bunch = []
            
            # Generate ray parameters for the 8x8 block
            for dy in range(8):
                for dx in range(8):
                    sx, sy = x + dx, y + dy
                    # Calculate ray origin and direction based on camera model [G]
                    origin = calculate_origin(sx, sy, image_width, image_height)
                    direction = calculate_direction(sx, sy, image_width, image_height)
                    
                    # Determine minimal distance to next sampling point
                    # and maximal ray length based on volume bounds [G]
                    min_dist = calculate_min_dist(origin, direction, volume_data)
                    max_len = calculate_max_len(origin, direction, volume_data)
                    
                    # Store in specific bit layout [S1]
                    # Starting Point: 3x24 bit, Min Dist: 3x16 bit, Max Len: 10 bit
                    ray_bunch.append({
                        "origin": origin,
                        "direction": direction,
                        "min_dist": min_dist,
                        "max_len": max_len
                    })
            
            # Download parameters to FPGA local storage
            # Overlaps with rendering time [S1]
            download_ray_parameters(ray_bunch, ray_param_ram)
            
            # Execute rendering algorithm on FPGA
            # Uses distance map to skip transparent regions [S1]
            rendered_block = execute_ray_casting(ray_bunch, volume_data, distance_map, transfer_function, reflectance_map, ray_step)
            
            # Upload results
            upload_results(rendered_block, image_width, image_height, x, y)

def calculate_origin(screen_x, screen_y, width, height):
    # Standard perspective/orthographic projection [G]
    return [0.0, 0.0, 0.0]

def calculate_direction(screen_x, screen_y, width, height):
    # Standard perspective/orthographic projection [G]
    return [0.0, 0.0, -1.0]

def calculate_min_dist(origin, direction, volume_data):
    # Compute distance to first voxel intersection [G]
    return 0.0

def calculate_max_len(origin, direction, volume_data):
    # Compute distance to last voxel intersection [G]
    return 100.0

def download_ray_parameters(ray_bunch, ram):
    # Transfer ray data from host to FPGA local storage [S1]
    pass

def execute_ray_casting(ray_bunch, volume_data, distance_map, transfer_function, reflectance_map, step):
    results = []
    for ray in ray_bunch:
        origin = ray["origin"]
        direction = ray["direction"]
        min_dist = ray["min_dist"]
        max_len = ray["max_len"]
        
        # Front-to-back compositing [G]
        alpha = 0.0
        color = [0.0, 0.0, 0.0]
        t = min_dist
        
        while t < max_len:
            # Use distance map to skip transparent regions [S1]
            dist_to_next = distance_map.sample(origin + direction * t)
            if dist_to_next == 0:
                t += step
                continue
            
            # Step through transparent region if distance is large
            step_size = min(step, dist_to_next)
            pos = origin + direction * t
            
            # Sample volume and apply transfer function [S1]
            density = volume_data.sample(pos)
            opacity, rgb = transfer_function.sample(density)
            
            # Compositing equation [G]
            alpha_new = alpha + (1.0 - alpha) * opacity
            color = color + (1.0 - alpha) * opacity * rgb
            alpha = alpha_new
            
            t += step_size
            
        results.append([color[0], color[1], color[2], alpha])
    return results

def upload_results(results, width, height, start_x, start_y):
    # Sends the final pixel values back to the host [S1]
    pass
```

**Parameters and Typical Values:**
*   **Backplane Ports:** High-speed I/O of 1 GB/s @ 66 MHz, 2 $\times$ 64-bit [S1].
*   **Host Interface:** PCI with two independent DMA channels, max throughput 125 MB/s [S1].
*   **M-Link:** FIFO-like point-to-point link, 80 MB/s LVDS implementation (similar to CERN S-Link) [S1].
*   **Multithreading Granularity:** 64 rays per bunch ($8 \times 8$) [S1].
*   **Reconfiguration Time:** 35 ms to 140 ms per mode [S1].

**Edge Cases and Pitfalls:**
*   **Reconfiguration Latency:** The need to reconfigure FPGAs for different modes introduces significant latency (35-140 ms) [S1]. This must be managed by overlapping configuration or task execution where possible, though the sources do not specify a mechanism for overlapping configuration *with* rendering, only parameter downloading [S1].
*   **Memory Bottleneck:** Performance is limited by memory speed (typically 100 MHz for FPGA devices), which is slower than contemporary SDRAM (150 MHz) or RDRAM (500 MHz) [S1]. This limits the potential throughput compared to ASICs if memory bandwidth were not the constraint [S1].
*   **Data Transfer Limits:** The PCI host interface is limited to 125 MB/s, which may become a bottleneck for transferring large volume datasets or high-resolution results [S1].

## Operating characteristics
*   **Performance:** The system achieves rendering rates of 30 Hz for $256^3$ data sets [S1].
*   **Comparison to ASICs:** FPGAs perform well relative to ASICs because the primary bottleneck is memory speed (limited to 100 MHz for most FPGA devices) rather than logic speed [S1]. Since SDRAMs can reach 150 MHz and RDRAMs up to 500 MHz, ASICs could theoretically offer higher performance if memory bottlenecks were resolved [S1].
*   **Complexity:** The architecture minimizes hardware complexity by reusing FPGAs for different tasks via reconfiguration [S1].
*   **Scalability:** The use of a subboard for volume memory allows integration of the memory read-out stage directly into the rendering pipeline [S1].
*   **Limitations:** The reliance on PCI for host communication (125 MB/s) and the reconfiguration latency are significant constraints [S1].

## Background
The ATLANTIS architecture was described in the paper "ATLANTIS: A hybrid approach combining the power of FPGA and RISC processor based on CompactPCI" by A. Kugel, H. Singpiel, J. Hesser, and R. Manner, presented at FPGA '99 in Monterey, CA, in 1999 [S1]. It was implemented on the ATLANTIS computing board, a versatile hardware platform for volume rendering applications [S1]. The design was influenced by earlier work such as VIRIM (1995) and the use of 3D distance transforms for ray casting [S1].

## Sources
[S1] Volume Rendering FPGA Hardware, https://doi.org/10.5753/sbac-pad.1999.19799, fetched 2026-10-24
