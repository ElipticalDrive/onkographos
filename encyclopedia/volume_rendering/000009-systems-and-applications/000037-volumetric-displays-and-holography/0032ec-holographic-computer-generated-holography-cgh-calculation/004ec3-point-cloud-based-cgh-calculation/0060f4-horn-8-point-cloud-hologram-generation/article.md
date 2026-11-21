# HORN-8 Point-Cloud Hologram Generation

Also called: HORN-8 algorithm, HORN-8 pipeline

## Overview
HORN-8 is a hardware-accelerated method for generating computer-generated holograms (CGH) from point-cloud data using a dedicated FPGA pipeline [S1]. It solves the computational bottleneck of point-cloud CGH by employing a normalized Fresnel approximation that enables integer arithmetic and incremental phase calculation, thereby eliminating the need for memory-intensive look-up tables for trigonometric operations [S1]. Unlike the layer method, which relies on FFTs and is unsuitable for hardware implementation, HORN-8 is optimized for localized models and achieves high parallelization through a memoryless pipeline architecture [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N_{obj}$ object points, where each point $j$ has a depth $Z_j$. The hologram plane has dimensions $N_{hol}$ pixels. The system supports a maximum of 65,536 object points per calculation block [S1].
*   **Output:** A hologram intensity map $I(X_\alpha, Y_\alpha)$ for all pixels $\alpha$ on the acquisition plane [S1].

**Data Structures**
The system is implemented on an FPGA with a specific pipeline structure [S1]:
*   **FPGA Board:** Contains 7 parallel pipelines [S1].
*   **Pipeline:** Consists of 1 Basic Processing Unit (BPU) and 639 Additional Processing Units (APU) [S1].
*   **Total Cores:** Each board has $640 \times 7 = 4,480$ parallel calculation circuits (cores) [S1].
*   **Memory:** The design is memoryless for the calculation pipeline itself, using Block RAM only for input/output data buffering [S1].

**Formulas**
The core calculation uses the normalized Fresnel approximation [S1]. The intensity $I(X_\alpha, Y_\alpha)$ at pixel $(X_\alpha, Y_\alpha)$ is the sum of contributions from all object points [S1]:

$$
I ( X_\alpha , Y_\alpha ) = \sum_{j=1}^{N_{obj}} \cos \left[ 2\pi \left( \frac{p}{2\lambda Z_j} (X_\alpha^2 + Y_\alpha^2) \right) \right] = \sum_{j=1}^{N_{obj}} \cos (2\pi \theta_{\alpha j})
$$

Where:
*   $p$ is the hologram pixel pitch [S1].
*   $\lambda$ is the wavelength of the reference light [S1].
*   $Z_j$ is the depth of object point $j$ [S1].
*   $X_\alpha, Y_\alpha$ are the pixel coordinates normalized by the pixel pitch $p$ (i.e., integer indices) [S1].
*   $A_j$ (intensity of point $j$) is set to 1 and excluded from the equation [S1].
*   $\theta_{\alpha j}$ is the normalized phase term for point $j$ at pixel $\alpha$: $\theta_{\alpha j} = \frac{p}{2\lambda Z_j} (X_\alpha^2 + Y_\alpha^2)$ [S1].

**Parameters**
*   **Operating Frequency ($f$):** 0.25 GHz [S1].
*   **Parallel Circuits ($C$):** 4,480 per board [S1].
*   **Max Points per Block:** 65,536 ($2^{16}$) [S1].

**Algorithm Steps**
The calculation proceeds pixel by pixel across the hologram plane. For each pixel, the contributions of all active object points are summed [S1].

1.  **Initialization (BPU):** For the first pixel $(X_1, Y_1)$, the Basic Processing Unit (BPU) calculates the initial phase $\theta_{1j}$ and the corresponding cosine value for each object point $j$ [S1].
2.  **Increment Calculation:** Compute the phase difference increment $\Delta_j$ for each object point $j$. This value is constant for a given object point and depth [S1]:
    $$
    \Delta_j = \frac{p}{2\lambda Z_j}
    $$
3.  **Phase Update (APU):** For subsequent pixels, the Additional Processing Units (APU) update the phase incrementally. The difference $\Gamma$ between adjacent pixels is summed to create the next phase value [S1]. The text defines this difference logic as $\Gamma = 2\Delta X_\alpha$ (referring to the change in the squared coordinate term or specific hardware increment logic) [S1]. The APU sums this difference to update the phase for the next pixel [S1].
4.  **Cosine Approximation:** Calculate the cosine of the updated phase using a novel approximate algorithm implemented with two adder-subtractor gates [S1]. This avoids memory look-up tables, allowing the pipeline to be memoryless [S1].
5.  **Summation:** Sum the cosine contributions from all object points to determine the hologram pixel intensity [S1].
6.  **Iteration:** Repeat steps 3-5 for all pixels in the hologram plane [S1].
7.  **Block Processing:** If the number of object points exceeds 65,536, divide the object into blocks of up to 65,536 points each and process them sequentially or in parallel across multiple boards [S1].

**Pseudocode**
The following pseudocode represents the logic of a single pipeline core (BPU or APU) processing one object point's contribution to the hologram plane. Note that in the actual hardware, these operations are parallelized across 4,480 cores per board [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def horn8_pipeline(object_points, hologram_width, hologram_height, p, lambda_wavelength, f, C):
    """
    Simulates the HORN-8 pipeline logic for a single object point contribution.
    In hardware, this is parallelized across many cores.
    """
    # Constants
    # p: pixel pitch
    # lambda_wavelength: wavelength
    # object_points: list of depths Z_j
    
    # Pre-calculate constant increment for this object point
    # Delta_j = p / (2 * lambda * Z_j)
    # Note: In hardware, this is fixed per point
    
    hologram = [[0.0 for _ in range(hologram_height)] for _ in range(hologram_width)]
    
    for x in range(hologram_width):
        for y in range(hologram_height):
            # Normalized coordinates (integers)
            X = x
            Y = y
            
            # Calculate phase theta for this pixel and object point
            # theta = (p / (2 * lambda * Z)) * (X^2 + Y^2)
            # This is done incrementally in hardware, but here we compute directly for clarity
            # However, the hardware uses:
            # 1. BPU for first pixel
            # 2. APU for subsequent pixels using difference Gamma
            
            # For implementation completeness, we use the direct formula
            # as the incremental logic is hardware-specific (add/sub gates)
            intensity_sum = 0.0
            
            for point in object_points:
                Z = point['z']
                if Z == 0:
                    continue # Avoid division by zero
                
                # Normalized phase term
                theta = (p / (2 * lambda_wavelength * Z)) * (X**2 + Y**2)
                
                # Cosine approximation using adder-subtractors (hardware specific)
                # Here we use standard math for the 'output' of the approximation
                cos_val = math.cos(2 * math.pi * theta)
                
                intensity_sum += cos_val
            
            hologram[x][y] = intensity_sum
            
    return hologram

def theoretical_time(N_obj, N_hol, f, C):
    """
    Calculates the theoretical time for HORN-8 generation.
    """
    return (N_obj * N_hol) / (f * C)
```

**Edge Cases and Pitfalls**
*   **Memory Bottleneck:** Previous systems used look-up tables for cosine operations, which consumed significant Block RAM (up to 94% usage) and limited parallelism [S1]. HORN-8 avoids this by using an approximate algorithm with adder-subtractors, keeping slice usage at 98% and Block RAM at 94% but freeing up memory for I/O [S1].
*   **Object Size Limit:** The maximum number of object points per calculation block is 65,536. Larger objects must be divided into blocks [S1].
*   **Precision:** The use of integer arithmetic and approximate cosine algorithms may introduce quantization errors, but the design prioritizes speed and hardware efficiency [S1].

## Operating characteristics
**Time and Memory Complexity**
*   **Time Complexity:** The theoretical calculation time for one board is $T_{horn} = \frac{N_{obj} \times N_{hol}}{f \times C}$ [S1].
*   **Memory Complexity:** The pipeline is memoryless for calculation, using Block RAM only for input/output data buffering [S1].

**Measured Performance**
*   **Single Board:** Generating a 1,920x1,080 hologram (2 million pixels) from 10,000 points took 0.019 seconds (155 fps) [S1]. This is 160x faster than a CPU (Core i7-6700K, 2.951 seconds) and 6x faster than a GPU (GTX TITAN X, 0.106 seconds) [S1].
*   **Cluster Performance:** An 8-board cluster generated a 100-million-pixel hologram from 10,000 points in 100 ms (10 fps) [S1].
*   **Large-Scale Generation:** Generating a 100-million-pixel hologram from 10-million points took 125 seconds on an 8-board cluster, with the object divided into 160 blocks of ~65,000 points each [S1].

**Scaling**
*   Performance scales linearly with the number of parallel circuits (cores) and boards [S1].
*   The system is designed to output hologram data in parallel with calculation, concealing transfer time $T_{out}$ [S1].

**Strengths and Limitations**
*   **Strengths:** High speed, suitable for hardware implementation, memoryless pipeline, effective for localized models (simulations, medical data) [S1].
*   **Limitations:** Requires division of large objects into blocks, approximate cosine algorithm may have precision trade-offs, not suitable for complete spaces (real scenes) which are better handled by the layer method [S1].

## Background
HORN-8 was developed as a next-generation holographic imaging system to demonstrate real-time generation of large-scale holograms [S1]. It builds on the point-cloud based CGH calculation method, which is effective for localized models and suitable for hardware implementation due to its independence from FFTs [S1]. The system was designed to overcome the memory and speed limitations of previous HORN computers by implementing a memoryless pipeline with a novel cosine approximation algorithm [S1].

## Sources
[S1] High-performance parallel computing for next-generation holographic imaging, https://doi.org/10.1038/s41928-018-0057-5, fetched 2026-10-29
