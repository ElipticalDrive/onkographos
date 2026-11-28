# Tracing-based Automatic Differentiation with Dynamic Compilation

Also called: Hybrid AD, JIT-traced AD

## Overview
This method is an automatic differentiation (AD) approach that ingests computation via tracing (Wengert tape) but combines it with dynamic compilation and function-level differentiation to produce output resembling source transformation AD [S1]. It solves the problem of high compilation overhead and poor runtime performance in dynamic dispatch scenarios common in differentiable rendering by dynamically specializing differential algorithms to the scene and problem statement while removing redundant steps introduced by the AD transformation [S1]. Unlike pure source transformation AD, it avoids committing loop state to memory at every iteration and reduces backend compilation time and register allocation costs by generating indirect calls to separate subroutines rather than merging all functions into a single large kernel body [S1].

## How it works
The method operates by tracing at two levels simultaneously: the AD layer traces differentiable computation, and the underlying JIT compiler traces the resulting derivative-related computation [S1].

**Inputs and Outputs:**
*   **Input:** A primal computation graph (Wengert tape) containing primal variables and their dependencies.
*   **Output:** A compiled residue of ordinary computation (machine code or optimized IR) ready for execution as part of a (mega-) kernel.

**Data Structures:**
*   **Wengert Tape:** A computation graph that captures operations for subsequent derivative propagation. It associates every primal variable $x$ with a differential version $\bar{x}$ (or $\delta x$) and records inter-variable dependencies [S1].
*   **JIT Intermediate Representation (IR):** The representation used by the underlying JIT compiler to trace the derivative-related computation residue [S1].

**Steps:**
1.  **Trace Primal and Differential Variables:** Associate every primal variable $x$ with a differential version $\bar{x}$ and record inter-variable dependencies on the Wengert tape [S1].
2.  **Traverse Tape for Derivatives:** Traverse the tape to compute differentiable variables. This process produces a flurry of operations referencing primal program state [S1].
3.  **Trace Residue with JIT:** The derivative-related computation generated in Step 2 is itself traced by the underlying JIT compiler. This step effectively destroys the AD data structures (the tape), leaving a residue of ordinary computation [S1].
4.  **Compile and Execute:** The residue is traced by the JIT compiler one level below so that it can run as part of a (mega-) kernel at a later point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode illustrating the two-level tracing process
# Note: This is a conceptual representation of the Dr.Jit mechanism described in [S1].

def compute_with_ad(primal_input):
    # Step 1: Trace primal computation and associate differentials
    # The Wengert tape records dependencies between primal variables x
    # and their differential versions dx.
    primal_output = primal_function(primal_input)
    
    # Step 2: Traverse the tape to compute differentiable variables
    # This produces operations referencing primal program state.
    # The AD system generates a computation graph for the derivatives.
    derivative_ops = traverse_tape(primal_output)
    
    # Step 3: Trace the derivative-related computation with JIT
    # The underlying JIT compiler traces the residue of ordinary computation.
    # The AD data structures (tape) are destroyed or consumed here.
    jit_ir = jit_compiler.trace(derivative_ops)
    
    # Step 4: Compile and execute the residue
    # The residue runs as part of a (mega-) kernel.
    compiled_kernel = jit_compiler.compile(jit_ir)
    result = compiled_kernel.execute()
    
    return result

# Example of checkpointing support mentioned in [S1]
def checkpointed_loop():
    data = initial_data
    for i in range(1000):
        data = f(data) # 'f' represents a potentially complex transformation
        dr.eval(data)  # Evaluate and trace
    dr.backward(data) # Backpropagate through all steps
```

**Parameters and Typical Values:**
*   **Tape Retention:** The destruction of the AD tape is immaterial if multiple AD traversals are needed; the tape is often retained in such cases [S1].
*   **Checkpointing:** The system supports checkpointing without intentional effort, as shown in unrolled loops with intermediate evaluation [S1].

**Edge Cases and Pitfalls:**
*   **Multiple Traversals:** If multiple AD traversals are required, the tape must be retained rather than destroyed [S1].
*   **Developer Responsibility:** The method does not free the developer from thinking about subtle details of the differentiation process. It facilitates but does not automate the introduction of physical and mathematical "reversibilities" [S1].

## Subtopics

### [JIT Compilation Pipeline](11647-jit-compilation-pipeline.md)
The JIT Compilation Pipeline is a computational substrate that captures Python or C++ code execution into an Intermediate Representation (IR) for later compilation into machine code, supporting asynchronous execution on CPU or GPU. It solves the problem of high compilation overhead and poor runtime performance in dynamic dispatch scenarios common in differentiable rendering. It sets itself apart from alternatives like JAX/XLA by generating indirect calls to separate subroutines rather than merging all functions into a single large kernel body or committing loop state to memory at every iteration, thereby reducing backend compilation time and register allocation costs.

## Operating characteristics
*   **Time Complexity:** The method reduces backend compilation time and register allocation costs by generating indirect calls to separate subroutines rather than merging all functions into a single large kernel body [S1].
*   **Memory Complexity:** The AD data structures are destroyed during the tracing of the residue, leaving only the ordinary computation residue in memory, which reduces memory overhead compared to retaining full AD graphs during execution [S1].
*   **Performance:** The approach enables dynamic specialization of differential algorithms to the scene and problem statement, removing redundant steps introduced by the AD transformation [S1].
*   **Strengths:** Combines the flexibility of tracing AD with the performance of source transformation AD; supports checkpointing naturally; reduces compilation overhead.
*   **Limitations:** Requires careful handling of tape retention for multiple traversals; does not automate the introduction of physical and mathematical "reversibilities" [S1].
*   **When to Use:** Suitable for differentiable rendering scenarios where dynamic dispatch and scene-specific specialization are beneficial, and where compilation overhead is a concern.
*   **When Not to Use:** Not suitable if the developer requires full automation of differentiation details or if multiple AD traversals are needed without retaining the tape.

## Background
This method is based on the Dr.Jit system, a Just-In-Time Compiler for Differentiable Rendering [S1]. It lies between tracing and source transformation AD, ingesting computation using tracing but producing output resembling source transformation AD through the combination with dynamic compilation and function-level differentiation [S1]. The approach is not unique to Dr.Jit and also underlies other hybrid AD systems, such as CppADCodeGen [S1]. The system was introduced in the paper "Dr.Jit: A Just-In-Time Compiler for Differentiable Rendering" [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
