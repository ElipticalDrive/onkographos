# Megakernel Generation

Also called: Megakernels, Megakernels vs Wavefronts

## Overview
Megakernel generation is a code generation strategy in differentiable rendering that consolidates all program instructions required to evaluate an integrand into a single, large GPU kernel [S1]. It solves the problem of excessive memory bandwidth consumption and storage overhead caused by inter-kernel communication in partitioned execution models [S1]. Unlike wavefront-based approaches that split computation into smaller kernels with intense data exchange through device memory, megakernels keep Monte Carlo integration fully contained within one kernel, storing most sample state in registers to minimize memory usage [S1].

## How it works
**Inputs:** A computational graph representing the rendering integrand, including branching logic for material polymorphism and Monte Carlo sampling logic.
**Outputs:** A single compiled GPU kernel (e.g., PTX/SASS) containing the full evaluation logic.

**Data Structures:**
*   **Register State:** The primary data structure for sample state (e.g., ray position, direction, accumulated throughput, material parameters) is kept in GPU registers [S1].
*   **Kernel Code:** A monolithic function containing all conditional branches (`if/else`) and function calls required for the integrand evaluation [S1].

**Steps:**
1.  **Partitioning Decision:** Determine that the Monte Carlo integration for a given sample should occur within a single kernel execution rather than being split across multiple passes [S1].
2.  **Code Generation:** Generate a single kernel function that includes all necessary instructions to evaluate the integrand, including dynamic dispatch for material types (subtype polymorphism) [S1].
3.  **State Management:** Assign variables representing the large amount of program state (millions of Monte Carlo samples) to registers rather than global memory [S1].
4.  **Compilation:** Compile the generated code into a GPU executable [S1].

**Formulas:**
None explicitly defined in the sources. The core concept is defined by the containment of integration:
$$ \text{Integration} \subset \text{Single Kernel} $$
Where the cost of inter-kernel communication is avoided [S1].

**Parameters:**
*   **Parallelization:** The simulation parallelizes over millions of Monte Carlo samples [S1].
*   **Kernel Size:** Determined by the complexity of the integrand and the available register file size per thread [S1].

**Edge Cases and Pitfalls:**
*   **Branch Divergence:** Megakernels contain large amounts of code connected via branch instructions. Branch divergence can reduce the effectiveness of vectorized program execution [S1].
*   **Register Pressure:** High register usage interferes with the GPU's latency-hiding mechanism, potentially reducing occupancy [S1].
*   **Size Limits:** There is a point where a megakernel is simply too large to run reasonably on a GPU [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Conceptual representation of a Megakernel
# Note: Actual implementation depends on the JIT compiler (e.g., Dr.Jit)
# and target GPU architecture.

def megakernel_eval(sample_index):
    """
    Evaluates the full integrand for a single Monte Carlo sample
    within a single kernel execution.
    """
    # 1. Load sample state into registers (implicit in JIT compilation)
    ray = get_ray(sample_index)
    throughput = 1.0
    throughput = throughput * evaluate_material(ray) # Includes branching for material type
    
    # 2. Perform Monte Carlo integration steps
    # All steps (bouncing, sampling, etc.) are inlined or called within this kernel
    # to avoid writing intermediate state to global memory
    for bounce in range(max_bounces):
        if not is_valid(ray):
            break
        
        # Dynamic dispatch for material (subtype polymorphism)
        material = get_material(ray)
        scattering = material.sample(ray)
        
        throughput *= scattering.throughput
        ray = scattering.next_ray
        
        # Accumulate contribution
        if is_surface_hit(ray):
            return throughput * evaluate_radiance(ray)
            
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
Megakernels were studied in the context of physically-based rendering and differentiable rendering. Laine et al. [2013] compared megakernels with wavefronts, finding wavefront execution to be superior at that time [S1]. However, nearly a decade and four hardware generations later, experiments using Dr.Jit indicated a shift back towards megakernels for the investigated workloads [S1]. The approach is closely tied to just-in-time (JIT) compilation systems that can dynamically generate specialized code for a given scene and optimization task [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
[S2] RoofGS: Roofline-Guided End-to-End Acceleration of 3D Gaussian Splatting, https://arxiv.org/abs/2608.15785, fetched 2026-10-29 (unofficial)
