# Host-Device Memory Transfer with Events

Also called: CUDA Event-based H2D/D2H Benchmarking

## Overview
This method provides a precise measurement pipeline for Host-to-Device (H2D) and Device-to-Host (D2H) memory transfer times, as well as kernel execution latency, using CUDA events [S1, unofficial]. It isolates transfer and compute phases to benchmark hardware throughput independently of CPU scheduling noise, distinguishing it from end-to-end timing or asynchronous streaming benchmarks [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Source host pointers (e.g., `h_sigma`, `h_source`, `h_I0`), destination device pointers (e.g., `d_sigma`, `d_source`, `d_I0`), kernel launch parameters, and output device pointers.
- **Outputs:** Elapsed times in milliseconds for H2D, kernel execution, and D2H phases; calculated throughput in GB/s and samples per second.

**Data Structures:**
- **CUDA Events:** Seven event handles are created to mark specific points in the execution stream: `e2e_start`, `e2e_stop`, `h2d_start`, `h2d_stop`, `kernel_start`, `kernel_stop`, `d2h_start`, `d2h_stop` [S1, unofficial].
- **Streams:** All operations are recorded on a single CUDA stream to ensure sequential ordering of events [S1, unofficial].

**Steps:**
1.  Create the seven CUDA events using `cudaEventCreate` [S1, unofficial].
2.  Record the start of the H2D transfer: `cudaEventRecord(h2d_start, stream)` [S1, unofficial].
3.  Perform H2D memory copies. The source cites copying `d_sigma`, `d_source`, and `d_I0` from their host counterparts [S1, unofficial].
4.  Record the stop of the H2D transfer: `cudaEventRecord(h2d_stop, stream)` [S1, unofficial].
5.  Record the start of the kernel execution: `cudaEventRecord(kernel_start, stream)` [S1, unofficial].
6.  Launch the kernel [S1, unofficial].
7.  Record the stop of the kernel execution: `cudaEventRecord(kernel_stop, stream)` [S1, unofficial].
8.  Record the start of the D2H transfer: `cudaEventRecord(d2h_start, stream)` [S1, unofficial].
9.  Perform D2H memory copy from `d_output` to `h_output_gpu` [S1, unofficial].
10. Record the stop of the D2H transfer: `cudaEventRecord(d2h_stop, stream)` [S1, unofficial].
11. Synchronize the end-to-end stop event to ensure all preceding operations on the stream are complete: `cudaEventSynchronize(e2e_stop)` [S1, unofficial]. Note: The source lists `e2e_stop` as a created event but does not explicitly show its recording step in the numbered list, implying it is recorded after `d2h_stop` or used as a synchronization barrier [S1, unofficial].
12. Calculate elapsed times using `cudaEventElapsedTime` for each phase: `h2d_ms`, `kernel_ms`, and `d2h_ms` [S1, unofficial].

**Formulas:**
- **H2D Throughput:**
  $$ \text{Throughput}_{H2D} = \frac{2.0 \times \text{sample\_bytes} + \text{ray\_bytes}}{1.0 \times 10^9 / (\text{h2d\_ms} / 1000.0)} \text{ GB/s} $$
  Where `sample_bytes` is the size of a single sample structure and `ray_bytes` is the size of ray data [S1, unofficial].
- **D2H Throughput:**
  $$ \text{Throughput}_{D2H} = \frac{\text{ray\_bytes}}{1.0 \times 10^9 / (\text{d2h\_ms} / 1000.0)} \text{ GB/s} $$
  [S1, unofficial].
- **Samples per Second:**
  $$ \text{Samples/sec} = \frac{\text{total\_samples}}{\text{kernel\_ms} / 1000.0} $$
  [S1, unofficial].

**Parameters:**
- `sample_bytes`: Size of the sample data structure in bytes [S1, unofficial].
- `ray_bytes`: Size of the ray data structure in bytes [S1, unofficial].
- `total_samples`: Total number of samples processed by the kernel [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Warm-up:** The source explicitly excludes warm-up runs from the benchmarking pipeline to avoid measuring hardware initialization overhead [S1, unofficial].
- **Synchronization:** `cudaEventSynchronize` is required to ensure accurate timing, as CUDA events are asynchronous with respect to the host [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import cuda

def benchmark_memory_transfer_and_kernel():
    # 1. Create events
    e2e_start = cuda.create_event()
    e2e_stop = cuda.create_event()
    h2d_start = cuda.create_event()
    h2d_stop = cuda.create_event()
    kernel_start = cuda.create_event()
    kernel_stop = cuda.create_event()
    d2h_start = cuda.create_event()
    d2h_stop = cuda.create_event()
    
    stream = cuda.create_stream()
    
    # 2. Record H2D start
    cuda.record_event(h2d_start, stream)
    
    # 3. Perform H2D copies
    # Assuming h_sigma, h_source, h_I0 are host pointers
    # Assuming d_sigma, d_source, d_I0 are device pointers
    cuda.memcpy_htod(d_sigma, h_sigma, stream)
    cuda.memcpy_htod(d_source, h_source, stream)
    cuda.memcpy_htod(d_I0, h_I0, stream)
    
    # 4. Record H2D stop
    cuda.record_event(h2d_stop, stream)
    
    # 5. Record Kernel start
    cuda.record_event(kernel_start, stream)
    
    # 6. Launch Kernel
    # launch_kernel(stream) # Implementation specific
    
    # 7. Record Kernel stop
    cuda.record_event(kernel_stop, stream)
    
    # 8. Record D2H start
    cuda.record_event(d2h_start, stream)
    
    # 9. Perform D2H copy
    # Assuming h_output_gpu is host pointer, d_output is device pointer
    cuda.memcpy_dtoh(h_output_gpu, d_output, stream)
    
    # 10. Record D2H stop
    cuda.record_event(d2h_stop, stream)
    
    # 11. Record E2E stop (implied by synchronization of e2e_stop)
    cuda.record_event(e2e_stop, stream)
    
    # 12. Synchronize
    cuda.synchronize_event(e2e_stop)
    
    # 13. Calculate times
    h2d_ms = cuda.elapsed_time(h2d_start, h2d_stop)
    kernel_ms = cuda.elapsed_time(kernel_start, kernel_stop)
    d2h_ms = cuda.elapsed_time(d2h_start, d2h_stop)
    
    return h2d_ms, kernel_ms, d2h_ms
```

## Subtopics

### Neural-module Latency Benchmark
A variant used to measure feed-forward neural-module latency on discrete GPUs (specifically an H20) [S2, unofficial]. It differs from the general transfer benchmark by focusing on end-to-end neural network inference latency rather than raw memory bandwidth [S2, unofficial]. The method executes one warmup run to stabilize hardware state, followed by five synchronized CUDA-event measurements to capture execution time [S2, unofficial]. It was tested on the ETH3D scene with 8 input views at resolutions 1022×756, 756×560, 504×378, and 378×280 [S2, unofficial]. Measured latencies were: 5.619 s (mean) at 1022×756, 2.533 s at 756×560, 1.032 s at 504×378, and 0.564 s at 378×280 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The measurement overhead is constant relative to the number of events, but the benchmarked operations (memory copy, kernel execution) dominate the time [S1, unofficial].
- **Memory Complexity:** Requires memory for 7 CUDA event handles and the data buffers being transferred [S1, unofficial].
- **Performance:** The method measures raw PCIe bandwidth and kernel execution speed. H2D throughput depends on `sample_bytes` and `ray_bytes` [S1, unofficial].
- **Strengths:** Isolates H2D, D2H, and kernel times, allowing precise bottleneck identification [S1, unofficial].
- **Limitations:** Requires synchronization for accurate timing, which can stall the CPU [S1, unofficial]. Does not account for PCIe transfer overheads that might be hidden in asynchronous streams [S1, unofficial].

## Background
The method originates from high-performance computing benchmarks for matrix multiplication and GPU kernels, specifically documented in the `hpc_foundational_demos` repository [S1, unofficial]. It is also applied in recent research on feed-forward 3D Gaussian Splatting (InfiniSplat v2) to benchmark neural module latency on discrete GPUs [S2, unofficial].

## Sources
[S1] wyh1309/hpc_foundational_demos, https://github.com/wyh1309/hpc_foundational_demos, fetched 2026-10-26 (unofficial)
[S2] InfiniSplat v2: Analyzing and Improving Feed-Forward 3DGS from Any Views, https://arxiv.org/abs/2610.20341, fetched 2026-10-29 (unofficial)
