# Time-Sliced Pipelined Architecture

Also called: Multi-core time-sliced pipeline, Time-slicing

## Overview
A hardware architecture for volume rendering where functional blocks are time-sliced to allow simultaneous processing of multiple tasks, enabling real-time rendering and continuous hardware-software communication [S1, unofficial]. It differs from standard linear pipelines by overlapping execution stages across independent sub-cores, allowing a single physical component to serve multiple logical stages or tasks via context switching [S1, unofficial].

## How it works
The architecture decomposes the rendering pipeline into independent, reusable functional blocks (sub-cores) that are configured or replicated based on processing requirements [S1, unofficial].

**Inputs and Outputs:**
- Input: Voxel data, classification parameters, gradient information, and transformation matrices.
- Output: Final pixel values for the frame buffer.

**Data Structures:**
- **Context RAM:** Stores intermediate results (e.g., dot products) for several clock cycles, allowing different components to access the same data without recomputation [S1, unofficial].
- **Normal Mode Registers:** Provide a common interface for communication between components [S1, unofficial].
- **Context Address Snooping Mechanism:** Allows components to access stored information in Context RAM [S1, unofficial].

**Functional Blocks:**
The pipeline is divided into the following blocks [S1, unofficial]:
1. Classifier
2. Gradient Calculator
3. Shearing-Warping Block (Shearer-Composer)
4. Shader (Shading Processor)
5. Composer

**Steps:**
1. **Decomposition:** Divide the rendering pipeline into the functional blocks listed above [S1, unofficial].
2. **Implementation:** Implement each block as an independent reusable component or sub-core [S1, unofficial]. These blocks are composed of basic functional units such as multipliers, adders, or inner product processors [S1, unofficial].
3. **Configuration:** Based on the type of processing required, the embedded sub-core is configured or replicated to execute a specific operation [S1, unofficial].
4. **Time-Slicing:** Use time-slicing to overlap execution stages. This allows simultaneous processing of several tasks by interleaving their execution on the hardware resources [S1, unofficial].
5. **Communication:** Components communicate via a common interface provided by Context RAM, normal mode registers, and the context address snooping mechanism [S1, unofficial]. For example, a calculated dot product is stored in a RAM cell and called up by another component (e.g., shader and composer) in subsequent cycles [S1, unofficial].

**Pitfalls:**
- **Context Switching Overhead:** The efficiency of time-slicing depends on the speed of context switching and the overhead of snooping mechanisms [S1, unofficial].
- **Resource Contention:** If multiple tasks require the same sub-core simultaneously, arbitration logic is needed [G].

## Subtopics

### Vg500 Parallel Pipeline Architecture
A specific implementation of the time-sliced pipelined architecture used in the VolumePro system [S2, unofficial]. It uses four parallel rendering pipelines to achieve real-time rendering rates by processing adjacent voxel values in the x-direction [S2, unofficial].

**Definition:**
The vg500 chip architecture uses four parallel rendering pipelines to achieve real-time rendering rates by processing adjacent voxel values in the x-direction [S2, unofficial].

**Steps:**
1. Pipelines work on adjacent voxel or sample values in the x-direction [S2, unofficial].
2. Each pipeline is connected to its neighbor by shift registers in each major unit, providing values to neighbors in the same cycle [S2, unofficial].
3. The shift registers of the rightmost pipeline recirculate values needed in the next cycle and send them to the leftmost pipeline [S2, unofficial].

**Parameters:**
- Clock frequency: 133 MHz [S2, unofficial].
- Each pipeline operates at 133 MHz and can accept a new voxel every cycle [S2, unofficial].
- Off-chip Voxel Memory supplies data to all pipelines [S2, unofficial].
- Pipelines write pixel values to an off-chip Pixel Memory [S2, unofficial].

**Pitfalls:**
- The shift registers of the rightmost pipeline must correctly recirculate values to the leftmost pipeline to maintain data continuity across the pipeline boundary [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The architecture enables real-time rendering by overlapping tasks. For the vg500 variant, four pipelines operating at 133 MHz can process approximately 532 million voxels per second, sufficient for rendering a $256^3$ dataset at 30 frames per second [S2, unofficial].
- **Memory Complexity:** Requires Context RAM for intermediate storage and off-chip Voxel/Pixel Memory. The vg500 chip includes 2 Mbits of on-chip SRAM [S2, unofficial].
- **Strengths:** Component reusability simplifies layout; time-slicing allows simultaneous processing of multiple tasks; continuous hardware-software communication supports interactive user interfaces [S1, unofficial].
- **Limitations:** Complexity in managing context switching and inter-pipeline communication (e.g., shift register recirculation) [S1, unofficial][S2, unofficial].

## Background
The time-sliced pipelined architecture was developed as part of the Shear-warp factorization volume rendering visualization system (VRS) [S1, unofficial]. It was designed to provide real-time rendering capabilities by leveraging component reusability and multi-core structures [S1, unofficial]. The vg500 chip, an ASIC implementation of this architecture, was developed for the VolumePro real-time ray-casting system [S2, unofficial].

## Sources
[S1] Shear-warp factorization volume rendering visualization system, https://archive.org/details/0162018299360, fetched 2026-10-26 (unofficial)
[S2] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
