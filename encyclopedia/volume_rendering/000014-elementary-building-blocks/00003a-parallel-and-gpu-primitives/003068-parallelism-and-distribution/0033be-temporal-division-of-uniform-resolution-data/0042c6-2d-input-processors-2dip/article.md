# 2D Input Processors (2DIP)
Also called: 2D Input Processing, 2DIP strategy
## Overview
2D Input Processors (2DIP) is a parallel I/O strategy for rendering large time-varying volume data that addresses the bottleneck where the data sending time ($T_s$) exceeds the rendering time ($T_r$) [S1]. Unlike [1D Input Processors (1DIP)](10828-1d-input-processors-1dip.md), which distributes a single time step across multiple input processors to reduce send time, 2DIP concurrently reads and distributes multiple time steps using a two-dimensional configuration of input processors [S1]. This ensures that rendering processors remain busy by overlapping I/O and preprocessing of future time steps with the rendering of the current one [S1].
## How it works
### Inputs and Outputs
- **Inputs**: Large time-varying volume dataset; number of rendering processors ($P_r$); estimated rendering time per frame ($T_r$); estimated total sending time for one time step ($T_s$); estimated total preprocessing time for one time step ($T_p$); estimated frame fetch time ($T_{f0}$) [S1].
- **Outputs**: A configuration of input processors organized into groups, and a stream of time steps delivered to rendering processors such that $T_{s0} < T_r$ [S1].

### Data Structures
The input processors are organized into a two-dimensional logical structure:
- **Groups**: $n$ groups of input processors.
- **Processors per Group**: $m$ input processors per group.
- **Total Input Processors**: $n \times m$ [S1].

Each group is responsible for reading, preprocessing, and distributing one complete time step of the data to all rendering processors [S1].

### Algorithm Steps
1. **Estimate Costs**: Determine $T_s$ (sending time for one time step), $T_p$ (preprocessing time for one time step), and $T_r$ (rendering time per frame) [S1].
2. **Calculate $m$**: Determine the number of input processors per group ($m$) to ensure the send time per group ($T_{s0}$) is less than the rendering time ($T_r$) [S1].
   $$ m \ge \frac{T_s}{T_r} $$
   [S1]
3. **Calculate $n$**: Determine the number of groups ($n$) required to keep the rendering processors busy. This is derived from the condition that the time to fetch and preprocess $n-1$ time steps equals the time to render $n$ frames (assuming $T_{s0} = T_r$ for the boundary case) [S1]:
   $$ n = \frac{T_{f0} + T_{p0}}{T_r} + 1 $$
   where $T_{p0} = T_p / m$ and $T_{f0}$ is the frame fetch time [S1].
   [S1]
   If we assume $T_{f0} \approx T_{s0}$ and $T_{p0} \approx T_p/m$, and set $T_{s0} = T_r$, the formula simplifies to:
   $$ n = \frac{T_s + T_p}{T_r} + 1 $$
   [S1]
4. **Configure Input Processors**: Organize the available input processors into $n$ groups of $m$ processors [S1].
5. **Concurrent Execution**:
   - Each group $i$ (where $0 \le i < n$) reads, preprocesses, and distributes time step $i$ to all rendering processors [S1].
   - With $m$ input processors working on one time step, the send time per group is:
     $$ T_{s0} = \frac{T_s}{m} $$
     [S1]
   - The preprocessing cost per group is spread across $m$ processors:
     $$ T_{p0} = \frac{T_p}{m} $$
     [S1]
   - The rendering processors render time step $i$ while input processors for group $i+1$ (if available) prepare time step $i+1$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Estimate times from profiling or historical data
T_s = estimate_send_time(dataset_size)
T_p = estimate_preprocess_time(dataset_size)
T_r = estimate_render_time(rendering_processors, resolution)
T_f0 = estimate_fetch_time() # Often approximated as T_s0 or T_s/m

# Calculate m: number of input processors per group
# Constraint: T_s0 < T_r => T_s / m < T_r => m > T_s / T_r
m = ceil(T_s / T_r)

# Calculate n: number of groups
# Constraint: Keep rendering busy. 
# Time to prepare n-1 steps = Time to render n steps (approx)
# (T_f0 + T_p0) * (n-1) = T_r * n ? 
# The source derives n = (T_f0 + T_p0) / T_r + 1 based on T_s0 = T_r boundary.
T_p0 = T_p / m
n = ceil((T_f0 + T_p0) / T_r) + 1

# Total input processors needed
total_input_processors = n * m

# Configuration
# Input processors are divided into n groups of m processors.
# Group i is responsible for time step i.

def input_processor_group(group_id, time_step_id, data, m_processors):
    """
    Each group reads, preprocesses, and distributes one time step.
    """
    # 1. Read data for time_step_id
    # Using Independent Contiguous Read is superior to Collective Noncontiguous Read [S1]
    raw_data = read_time_step(time_step_id)
    
    # 2. Preprocess data (spread across m processors)
    # Preprocessing cost T_p0 = T_p / m
    processed_data = preprocess(raw_data)
    
    # 3. Distribute to all rendering processors
    # Send time T_s0 = T_s / m
    distribute_to_renderers(processed_data)

def render_processor(render_id, time_step_id):
    """
    Renders the current time step.
    """
    # Wait for data from input processor group
    data = receive_from_input_group(time_step_id)
    render(data)
```

### Parameters and Typical Values
- **$m$**: Must be at least $\lceil T_s / T_r \rceil$. In tests with 256 rendering processors, $m=14$ was sufficient to completely hide I/O and preprocessing [S1].
- **$n$**: Depends on $T_{f0}$, $T_p$, and $T_r$. Typically $n > 1$ to allow pipelining [S1].
- **Hardware**: Tested with up to 256 rendering processors and 8+ input processors [S1].

### Edge Cases and Pitfalls
- **$T_s \le T_r$**: If sending time is less than rendering time, 1DIP is sufficient and more efficient. 2DIP is only needed when $T_s > T_r$ [S1].
- **Insufficient Input Processors**: If the total number of available input processors is less than $n \times m$, the pipeline stalls. The system must either reduce $n$ (fewer time steps in flight) or increase $m$ (if possible) to reduce $T_{s0}$ [S1].
- **Read Strategy**: Using Collective Noncontiguous Read is inferior to Independent Contiguous Read for 2DIP, resulting in higher sending and total times [S1].

## Operating characteristics
- **Time Complexity**: The send time per group is reduced to $T_{s0} = T_s / m$ [S1]. The total time for rendering $N$ frames is dominated by the rendering time $T_r$ once the pipeline is full, as I/O and preprocessing are hidden [S1].
- **Memory Complexity**: Requires storing $n$ time steps in the input processor groups' memory buffers simultaneously [S1].
- **Measured Performance**: With 8 or more input processors, total time drops below 10 seconds, and sending time drops below 1 second, enabling multiple frames per second [S1]. At 256 rendering processors, 2DIP with $m=14$ completely hides I/O and preprocessing cost, whereas 1DIP does not scale as well [S1].
- **Strengths**: Effectively hides I/O and preprocessing latency when $T_s > T_r$; enables interactive frame rates for large time-varying datasets [S1].
- **Limitations**: Requires a significant number of input processors ($n \times m$); less scalable than 1DIP if $T_s$ is not significantly larger than $T_r$ [S1].
- **When to Use**: Use when $T_s > T_r$ or when 1DIP limits the reduction of $T_s$ [S1].
- **When Not to Use**: Use 1DIP when $T_r > T_s$ [S1].

## Background
2D Input Processors were introduced by Hongfeng Yu, Kwan-Liu Ma, and Joel Welling in their 2004 paper "I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data" [S1]. The work was presented at the Eurographics Symposium on Parallel Graphics and Visualization (EGPGV) [S1]. The strategy was developed to address the I/O bottleneck in massively parallel rendering of large time-varying volume data, where the time to send data to rendering processors exceeded the time to render the image [S1]. It builds on the 1DIP strategy by adding a second dimension of parallelism (multiple time steps) to overcome the limitations of 1DIP when send times are high [S1].

## Sources
[S1] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
