# Tracing-based Automatic Differentiation with Dynamic Compilation

Also called: Hybrid AD, JIT-traced AD

## Overview
Tracing-based Automatic Differentiation with Dynamic Compilation is a hybrid differentiation technique that combines Wengert tape tracing with Just-In-Time (JIT) compilation to generate optimized machine code for derivative computation [S1]. It solves the performance and code-generation overhead issues of pure source transformation AD and the runtime interpretation overhead of pure tracing AD by dynamically specializing differential algorithms to the specific scene and problem statement [S1]. Unlike pure source transformation AD, it removes redundant steps introduced by the AD transformation at runtime; unlike pure tracing AD, it produces output resembling source transformation by compiling the derivative residue into a kernel [S1].

## How it works
The method operates by tracing differentiable computation at two simultaneous levels: the AD layer traces the primal and differential variables, while the underlying JIT compiler traces the resulting derivative-related computation [S1].

**Inputs and Outputs:**
*   **Input:** A differentiable program (primal computation) and a set of primal variables $x$.
*   **Output:** A compiled kernel (residue) that computes the derivatives, executed later.

**Data Structures:**
*   **Wengert Tape:** A computation graph that records inter-variable dependencies. Each primal variable $x$ is associated with a differential version $\bar{x}$ (or $\delta x$) [S1].
*   **JIT Intermediate Representation (IR):** Captures the derivative-related computation residue for compilation [S1].

**Steps:**
1.  **Trace Primal and Differential Variables:** Associate every primal variable $x$ with a differential version $\bar{x}$. Record inter-variable dependencies on the Wengert tape as operations are executed [S1].
2.  **Tape Traversal for Derivatives:** Traverse the tape to compute the differentiable variables. This process produces a flurry of operations that reference the primal program state [S1].
3.  **JIT Tracing of Residue:** The derivative-related computation generated in Step 2 is itself traced by the underlying JIT compiler. This step effectively "destroys" the AD data structures (the tape), leaving a residue of ordinary computation [S1].
4.  **Compilation and Execution:** The residue is compiled into a (mega-) kernel and executed at a later point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode illustrating the two-level tracing process
# Note: This is a conceptual representation of the Dr.Jit mechanism described in [S1].

def trace_and_compile(primal_func, inputs):
    """
    Simulates the tracing-based AD with dynamic compilation.
    """
    # Step 1: Initialize AD context and tape
    tape = WengertTape()
    # Associate primal variables with differential versions
    # dr.init(inputs) would create dual-like structures or tape entries
    
    # Step 2: Execute primal computation, recording on tape
    # The JIT compiler also traces this execution
    primal_output = primal_func(inputs)
    
    # Step 3: Traverse tape to generate derivative operations
    # This produces a "flurry of operations referencing primal program state"
    derivative_ops = traverse_tape(tape, primal_output)
    
    # Step 4: JIT compiler traces the derivative operations
    # "Differentiation destroys the AD data structures, which leaves a residue"
    # The tape is effectively consumed/destroyed here for the residue generation
    residue_ir = jit_compiler.trace(derivative_ops)
    
    # Step 5: Compile residue into kernel
    kernel = jit_compiler.compile(residue_ir)
    
    # Step 6: Execute kernel later
    # derivatives = kernel.execute()
    return kernel

# Example usage pattern from [S1]:
# dr.eval(data) # Traces primal
# dr.backward(data) # Traces derivative residue and compiles/executes
```

**Parameters and Edge Cases:**
*   **Tape Retention:** The destruction of the AD tape is immaterial if multiple AD traversals are needed; in such cases, the tape is often retained [S1].
*   **Checkpointing:** The system supports checkpointing naturally due to the separation of tracing and compilation, allowing for unrolled loops with intermediate evaluations (e.g., `dr.eval` inside a loop followed by `dr.backward`) [S1].
*   **Redundancy Removal:** The JIT compiler removes redundant steps introduced by the AD transformation, which is a key advantage over pure source transformation [S1].

## Subtopics
## Operating characteristics
*   **Performance:** The method dynamically specializes differential algorithms to the scene and problem statement, removing redundant steps [S1]. This avoids the costly tensorial optimizations of ML frameworks (PyTorch, TensorFlow, JAX) which are inefficient for the large amount of arithmetic in Path-Based Differentiable Rendering (PBDR) workloads [S1].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Combines the ease of use of tracing with the performance of source transformation; supports checkpointing without explicit implementation; removes AD-induced redundancy [S1].
*   **Limitations:** Does not automate the introduction of physical and mathematical "reversibilities"; the developer must still think about subtle details of the differentiation process [S1].
*   **When to use:** Suitable for PBDR workloads where dynamic specialization and kernel compilation are beneficial, and where the overhead of rich IR optimizations in ML frameworks is prohibitive [S1].

## Background
This approach is implemented in **Dr.Jit**, a Just-In-Time Compiler for Differentiable Rendering [S1]. It lies between tracing and source transformation AD, ingesting computation via tracing but producing output resembling source transformation through dynamic compilation and function-level differentiation [S1]. The technique of tracing derivative-related computation and destroying the AD data structures to leave a residue is not unique to Dr.Jit and underlies other hybrid AD systems such as CppADCodeGen [Leal and Bell 2017] [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
