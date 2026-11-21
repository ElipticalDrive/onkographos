# 1D Input Processors (1DIP)
Also called: 1D Input Processing, 1DIP, 1DIP strategy
## Overview
1D Input Processors (1DIP) is a parallel I/O strategy that uses multiple input processors to fetch and preprocess a single complete time step of volume data, maximizing parallel file system bandwidth [S1]. It differs from its sibling [2D Input Processors (2DIP)](10833-2d-input-processors-2dip.md) by focusing on parallelizing the read of one time step rather than concurrently reading multiple time steps, making it suitable when sending time per time step is the bottleneck [S1].
## How it works
**Inputs and Outputs:**
- Input: Raw volume data for a single time step stored on a parallel file system.
- Output: Preprocessed data blocks sent to rendering processors.

**Data Structures:**
- Each input processor maintains local buffers for the fetched time step data.
- Rendering processors maintain local buffers to merge incoming data into complete octree blocks [S1].

**Steps:**
1. **Fetch:** $m$ input processors concurrently read the single time step from the parallel file system. Each processor reads a distinct portion of the data to maximize bandwidth [S1].
2. **Preprocess:** Each input processor preprocesses its assigned portion. Preprocessing may include load balancing calculations and data quantization [S1].
3. **Send:** Input processors send the preprocessed data to rendering processors. The sending strategy is designed to minimize idle time for rendering processors [S1].
4. **Merge:** Each rendering processor merges incoming data from the input processors to form complete local octree blocks. This merge operation requires no inter-processor communication between rendering processors [S1].

**Formula:**
The optimal number of input processors $m$ is calculated to hide I/O and preprocessing costs behind rendering time [S1]:

$$ m = \frac{T_f + T_p}{T_s} + 1 $$

Where:
- $T_f$: Time to fetch the data from the file system [S1].
- $T_p$: Preprocessing time (including load balancing and quantization) [S1].
- $T_s$: Time to send the data to a rendering processor [S1].

If the sending time $T_s$ is smaller than the rendering time $T_r$, the formula can be adjusted to use fewer input processors while keeping rendering processors busy [S1]:

$$ m = \frac{T_f + T_p}{T_r} + 1 $$

**Parameters:**
- $T_f, T_p, T_s, T_r$: Measured in seconds. Typical values depend on hardware and data size. For example, on LeMieux with 64 rendering processors and 512x512 image resolution, $T_r \approx 2s$, $T_f \approx 12.32s$, $T_p \approx 9.06s$ (complex quantization) [S1].

**Edge Cases and Pitfalls:**
- If $T_s$ becomes larger than $T_r$, 1DIP becomes less scalable compared to 2DIP [S1].
- Preprocessing cost can vary significantly; using simpler quantization methods can reduce $T_p$ substantially [S1].
- The strategy assumes that the overhead of collective I/O is too high, making independent reads preferable [S1].

## Operating characteristics
- **Time Complexity:** The total time per time step is dominated by $\max(T_r, \frac{T_f + T_p}{m-1} + T_s)$ [S1]. With optimal $m$, I/O and preprocessing are hidden behind rendering [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On LeMieux (HP/Compaq AlphaServer, 3,000 processors) with 64 rendering processors and 512x512 image:
  - 1 input processor: I/O+preprocessing $\approx 22s$, rendering $\approx 2s$ [S1].
  - 12 input processors: I/O+preprocessing $\approx 2s$, matching rendering time [S1].
  - At 1024x1024 resolution, rendering time increases to 3.63s, requiring only 7 input processors to match total time [S1].
- **Scaling:** 1DIP is less scalable than 2DIP at high processor counts. With 256 rendering processors, 2DIP can hide I/O with $m=14$, but 1DIP cannot [S1].
- **Strengths:** Maximizes file system bandwidth for single time steps; effective when sending time is significant [S1].
- **Limitations:** Less scalable than 2DIP at high processor counts; requires careful tuning of $m$ [S1].
- **When to use:** When rendering time is significant and sending time per time step is a bottleneck, and when collective I/O overhead is high [S1].
- **When not to use:** When high processor counts make 2DIP more effective at hiding I/O costs [S1].

## Background
1DIP was introduced by Hongfeng Yu, Kwan-Liu Ma, and Joel Welling in their 2004 paper "I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data" [S1]. It was tested on the LeMieux system at the Pittsburgh Supercomputing Center [S1]. The strategy addresses the I/O problem in massively parallel rendering by employing multiple data servers to fetch a single time step [S1].

## Sources
[S1] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
