# Dr.Jit Optimizations

Also called: Global optimizations, Polymorphism-aware optimizations

## Overview
Dr.Jit Optimizations are a set of global transformations applied during the tracing phase to improve code generation, specifically addressing the opaque boundary of dynamic dispatch that breaks interprocedural optimizations in backends like LLVM and OptiX [S1]. Unlike ML-centric IR and tensorial optimizations which are costly for rendering, Dr.Jit's optimizations are tailored for the large amount of arithmetic in PBDR workloads [S1].

## How it works
**Inputs:** Traced computation graph containing polymorphic calls and variable references.
**Outputs:** Optimized intermediate representation (IR) ready for backend compilation.

**Data Structures:**
*   **Variable Reference Count:** Tracks the number of active references to each variable to identify dead code [S1].
*   **Value Numbering Map:** Maps identical computational results to a single canonical ID to enable deduplication [S1].

**Steps:**
1.  **Constant Propagation:** Automatically performed during tracing to replace variables with their constant values where possible [S1].
2.  **Dead Code Elimination:** Performed via variable reference counting. If a variable is unreferenced, it is removed [S1].
3.  **Unreferenced Output Removal:** The system removes unreferenced outputs from function calls. This triggers a cascade effect, removing unreferenced inputs and further reducing the surrounding program's computational load [S1].
4.  **Devirtualization:** Computation that is identical in all sub-traces is devirtualized. If all sub-traces output a variable with the same ID, that variable is moved out of the call [S1].
5.  **Trace and Merge:** The system traces every reachable instance of polymorphic functions (e.g., ubershader materials) without making assumptions about parameter subsets. The resulting identical functions are then merged into a single subroutine [S1].
6.  **Deduplication:** Built on value numbering. Identical traces are collapsed into a single subroutine to handle scenes with thousands of objects of the same type [S1].

**Pitfalls:**
*   **Type Equivalence vs. Operation Sequence:** Type equivalence does not imply identical operation sequences for complex materials. For example, Disney’s Principled BSDF may have different active subsets of parameters, procedural textures, or derivative tracking requirements depending on the instance [S1]. Dr.Jit avoids assumptions and traces every instance to ensure correctness [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_traced_program(program):
    # Step 1: Constant Propagation (automatic during tracing)
    program = propagate_constants(program)
    
    # Step 2: Dead Code Elimination via Reference Counting
    program = eliminate_dead_code(program)
    
    # Step 3: Remove Unreferenced Outputs and Inputs
    # This cascades to remove dependent inputs
    program = remove_unreferenced_io(program)
    
    # Step 4: Devirtualize Identical Sub-traces
    # Move variables out of calls if all sub-traces output same ID
    program = devirtualize_identical_calls(program)
    
    # Step 5: Trace Polymorphic Instances
    # Trace every reachable instance without assuming parameter subsets
    traced_instances = trace_polymorphic_instances(program)
    
    # Step 6: Deduplicate and Merge Identical Traces
    # Collapse identical traces into single subroutines
    program = merge_identical_traces(traced_instances)
    
    return program
```

## Operating characteristics
*   **Performance Impact:** Removing unreferenced computation, arguments, and return values improves performance by an average factor of 2.5x in the OptiX backend [S1].
*   **CPU Impact:** On the CPU, removing unreferenced computation has essentially no impact on performance, likely because modern superscalar CPUs can absorb redundant computation by issuing multiple instructions per clock cycle [S1].
*   **Compilation Time:** Subroutine deduplication reverts growth in compilation time caused by other optimizations and provides a speed boost by reducing divergence under vectorized execution [S1].
*   **Strengths:** Tailored for the large amount of arithmetic in PBDR workloads, avoiding the cost of ML-centric tensorial optimizations [S1]. Effective at handling scenes with thousands of identical objects [S1].
*   **Limitations:** The optimization strategy relies on tracing every instance, which can increase compilation time if deduplication is not effective [S1].

## Background
Dr.Jit Optimizations were introduced in the Dr.Jit library, designed for differentiable programming and volume rendering. The approach addresses the limitations of standard backend compilers (LLVM/OptiX) when faced with dynamic dispatch boundaries [S1]. The optimizations are specifically tailored for the arithmetic intensity of PBDR workloads, distinguishing them from optimizations designed for neural network computations [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
