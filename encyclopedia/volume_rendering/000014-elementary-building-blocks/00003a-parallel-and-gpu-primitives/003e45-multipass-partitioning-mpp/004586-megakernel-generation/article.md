# Megakernel Generation

Also called: Megakernels, Megakernels vs Wavefronts

## Overview
Megakernel generation is a code generation strategy in differentiable rendering that consolidates all program instructions required to evaluate an integrand into a single, large GPU kernel [S1]. It solves the problem of excessive memory bandwidth consumption and storage overhead caused by inter-kernel communication in partitioned execution models [S1]. Unlike wavefront-based approaches that split computation into smaller kernels with intense data exchange through device memory, megakernels keep Monte Carlo integration fully contained within one kernel, storing most sample state in registers to minimize memory usage [S1].

## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.

**Data Structures:**
*   **Register State:** The primary data structure for sample state (e.g., ray position, direction, accumulated throughput, material parameters) is kept in GPU registers [S1].
*   **Kernel Code:** A monolithic function containing all conditional branches and function calls required for the integrand evaluation [S1].

**Steps:**
1.  **Partitioning Decision:** Partition computation so that Monte Carlo integration occurs within a single megakernel rather than being split across multiple passes [S1].
2.  **State Management:** Store most sample state in registers instead of global memory [S1].
3.  **Communication Minimization:** Minimize inter-kernel communication by reducing storage and memory bandwidth costs associated with exchanging information through device memory [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
*   **Parallelization:** The simulation parallelizes over millions of Monte Carlo samples representing a large amount of program state [S1].
*   **Kernel Size:** Determined by the complexity of the integrand and the available register file size per thread [G].

**Edge Cases and Pitfalls:**
*   **Branch Divergence:** Megakernels contain large amounts of code connected via branch instructions, which can reduce the effectiveness of vectorized program execution [S1].
*   **Register Pressure:** High register usage interferes with the GPU's latency-hiding mechanism, potentially reducing occupancy [S1].
*   **Size Limits:** There is a point where a megakernel is simply too large to run reasonably on a GPU [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper definitions [G]
def get_ray_origin(idx): return [0.0, 0.0, 0.0]
def get_ray_direction(idx): return [0.0, 0.0, 1.0]
def trace_intersection(o, d): return type('Hit', (), {'valid': True, 'point': [0,0,0], 'normal': [0,0,1], 'material': None})()
def sample_material(mat, n, d): return [d, 1.0, 0.0]

def megakernel_eval(sample_index):
    # 1. Partition computation: keep Monte Carlo integration in one kernel
    # 2. Store sample state in registers
    ray_origin = get_ray_origin(sample_index)
    ray_direction = get_ray_direction(sample_index)
    throughput = 1.0
    max_bounces = 10
    
    for bounce in range(max_bounces):
        hit = trace_intersection(ray_origin, ray_direction)
        if not hit.valid:
            break
        
        material = hit.material
        sample_direction, sample_throughput, sample_emission = sample_material(material, hit.normal, ray_direction)
        throughput *= sample_throughput
        
        ray_origin = hit.point + hit.normal * 1e-4
        ray_direction = sample_direction
        
        if sample_emission > 0.0:
            return throughput * sample_emission
    return 0.0
```

## Subtopics

### Megakernel Fusion
Fuses preprocessing, tile duplication, and key generation into a single kernel to reduce intermediate data movement in the front end and reduce arithmetic overhead in rasterization [S2, unofficial]. It compresses spherical harmonics (SH) and geometric attributes, packs the tile index and quantized depth into a dynamic 32-bit sorting key, and performs these operations in one pass [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Lower memory bandwidth usage compared to wavefronts because intermediate state is kept in registers rather than written to and read from device memory [S1].
*   **Measured Performance:** Experiments indicate that for the workloads investigated, the balance has shifted back towards megakernels compared to wavefronts, despite earlier findings favoring wavefronts [S1].
*   **Strengths:** Reduces storage and memory bandwidth costs associated with exchanging information through device memory [S1]. Enables dynamic generation of specialized code for a given scene and algorithm [S1].
*   **Limitations:** Branch divergence reduces vectorized execution effectiveness [S1]. High register usage interferes with GPU latency-hiding [S1]. May become too large to run reasonably on a GPU [S1].
*   **When to Use:** When memory bandwidth is a bottleneck and the integrand can be contained within a single kernel without exceeding register limits [S1].
*   **When Not to Use:** When branch divergence is severe or register pressure is too high to maintain occupancy [S1].

## Background
Megakernels were studied in the context of physically-based rendering and differentiable rendering [S1]. Laine et al. [2013] compared megakernels with wavefronts, finding wavefront execution to be superior at that time [S1]. However, nearly a decade and four hardware generations later, experiments using Dr.Jit indicated a shift back towards megakernels for the investigated workloads [S1]. The approach is closely tied to just-in-time (JIT) compilation systems that can dynamically generate specialized code for a given scene and optimization task [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
[S2] RoofGS: Roofline-Guided End-to-End Acceleration of 3D Gaussian Splatting, https://arxiv.org/abs/2608.15785, fetched 2026-10-29 (unofficial)
