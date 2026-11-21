# Custom Hardware Units
Also called: Dedicated Volume Rendering Hardware, ASIC Volume Rendering, FPGA Volume Rendering

## Overview
Custom Hardware Units refer to volume rendering architectures that utilize dedicated, non-standard graphics hardware or specialized processors (ASICs, FPGAs, DSPs) rather than general-purpose CPUs or standard GPUs . This family addresses the demand for real-time volume rendering when general-purpose hardware lacked sufficient processing power for the highly regular computations involved . Unlike sibling methods that leverage existing graphics pipelines (such as texture slicing on GPUs) or specific software-accelerated frameworks, custom hardware relies on bespoke silicon designed specifically to offload geometry resampling, shading, and compositing .

## Shared principles
This family shares the principle of hardware specialization to overcome the computational bottlenecks of software-based rendering. While specific implementations vary (ASIC vs. FPGA vs. DSP), they all aim to decouple or parallelize the heavy geometric and shading loads . Common architectural goals include:
1.  **Dedicated Resampling Units:** Separate hardware blocks handle the high-speed generation of ray starting points and interpolation of volume data, often operating significantly faster than the shading units .
2.  **Specialized Memory Architectures:** Use of on-chip buffers, FIFOs, or precalculated weight memories to manage bandwidth constraints and ensure conflict-free access to volume data .
3.  **Fixed-Point or Low-Precision Arithmetic:** Many custom units (e.g., VIRIM, Cube) utilize 16-bit precision for density and gradient values to maximize throughput and minimize hardware complexity .
4.  **Pipeline Parallelism:** Data flows through distinct stages (resampling $\rightarrow$ buffering $\rightarrow$ shading/compositing) often with asynchronous buffering to hide latency .

## Subtopics

### [Hardware-based Geometry Unit Resampling](3139-hardware-based-geometry-unit-resampling.md)
Implements a dedicated ASIC geometry unit for perspective calculations and resampling. It solves the high cost of geometry operations (consuming ~80% of software time) by using specialized logic for address generation and interpolation, achieving up to 640 MB/s bandwidth. Distinguished by precalculated interpolation weights and Sobel filter-based gradient estimation within the VIRIM architecture.

### [Hardware-based Direct Volume Ray Casting](5442-hardware-based-direct-volume-ray-casting.md)
Offloads ray-casting and shading to programmable DSPs. It solves the CPU bottleneck by decoupling fast geometry resampling from slower shading computations using a FIFO buffer. Focuses on image quality and flexible illumination (shadows) via object-order ray casting, utilizing a custom memory architecture.

### [Custom Hardware Rendering](5746-custom-hardware-rendering.md)
Refers to direct volume rendering via dedicated, non-standard graphics hardware like the Cube and VolumePro systems. It addresses the need for real-time rendering by using voxel processors that traverse data in memory order and sample processors for compositing. Unlike siblings, it relies on bespoke silicon pipelines rather than reconfigurable logic or DSP arrays.

### [ATLANTIS FPGA Volume Rendering Architecture](16510-atlantis-fpga-volume-rendering-architecture.md)
Uses FPGA reconfiguration to define special-purpose processors for distinct pipeline stages, minimizing hardware overhead. Distinguishes itself by dynamically reloading configurations for initialization, distance calculation, and rendering, leveraging a hybrid FPGA/RISC approach on CompactPCI. Utilizes 3D distance transforms for transparent region extension.

## Background
The development of custom hardware units was driven by the limitations of general-purpose CPUs and early GPUs in the 1990s and early 2000s, which could not achieve real-time frame rates for large volumetric datasets . Key architectures include the VIRIM system (Voxel Image Rendering on Integrated Modules), which utilized ASICs for geometry and DSPs for shading , and the Cube system, which employed eight identical rendering pipelines . The ATLANTIS architecture later explored Field-Programmable Gate Arrays (FPGAs) to offer reconfigurability . These systems were primarily used in medical imaging and scientific visualization where high-fidelity, real-time interaction was critical .

## Sources
[S1] Custom Hardware Units notes
[S2] Hardware-based Direct Volume Ray Casting notes
