# Custom Adjoint Interface

Also called: dr.CustomOp

## Overview
The Custom Adjoint Interface is a mechanism in differentiable programming systems that allows users to define custom forward and reverse-mode derivatives for arbitrary functions, extending standard automatic differentiation (AD) by tracking implicit dependencies on global state [S1]. It distinguishes itself from standard AD interfaces by explicitly monitoring variable accesses during primal execution to register implicit inputs and outputs, ensuring correct gradient flow for functions interacting with global scene parameters or closures [S1].

## How it works
The interface operates by intercepting function calls during AD traversal and invoking user-provided or system-generated callbacks to handle derivative propagation.

### Inputs and Outputs
- **Inputs**: The primal function arguments, return values, and implicitly accessed global variables (e.g., texture parameters, BRDF coefficients) [S1].
- **Outputs**: Gradients with respect to explicit arguments and implicit global variables [S1].

### Data Structures
The system maintains a computational tape. When a `CustomOp` is encountered, the AD system references intermediate steps and temporaries produced by the primal function, which must be recomputed if needed for the backward pass [S1].

### Steps
1. **Primal Execution**: The user-defined function executes. During this phase, the system monitors all variable accesses [S1].
2. **Implicit Dependency Registration**: Any variable accessed from the surrounding environment (e.g., closures, global state) is automatically registered as an implicit dependence with the AD system [S1].
3. **Forward Pass**: If the forward derivative is defined, the `forward()` callback is invoked to compute the primal output and potentially intermediate values [S1].
4. **Backward Pass**: If the reverse derivative is required, the `backward()` callback is invoked. This callback converts output derivatives into input derivatives [S1].
5. **Gradient Aggregation**: For methods with implicit reads (global state dependencies), the system issues atomic scatter-reductions to update the gradients of scene parameters [S1].

### Pseudocode
The following Python-like pseudocode illustrates the dynamic generation of Vector-Jacobian Product (VJP) functions for polymorphic calls, a key feature of the interface [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def func_vjp(grad_out, arg_1, arg_2, ..):
    # Enable gradient tracking for inputs
    dr.enable_grad(arg_1, arg_2, ..)
    
    # Re-run the primal function to capture dependencies
    result = func(arg_1, arg_2, ..)
    
    # Set the gradient of the output to the incoming adjoint
    dr.set_grad(result, grad_out)
    
    # Backpropagate through the primal computation
    dr.backward(result)
    
    # Return the computed gradients for the inputs
    return dr.grad(arg_1), dr.grad(arg_2), ..
```

### Parameters and Edge Cases
- **Implicit Reads**: Functions that depend on global state must handle atomic scatter-reductions for gradient updates [S1].
- **Zero-Valued Derivatives**: If a return value does not carry derivatives, the system devirtualizes the zero-valued derivative and propagates zero-valued input derivatives to simplify call targets using constant propagation [S1].
- **Polymorphism**: Dynamic VJP generation is used for polymorphic method calls, where the JIT layer traces each possible target to materialize differentiation into ordinary traced computation [S1].

## Subtopics

### [GradBlock with Automatic Differentiation](11540-gradblock-with-automatic-differentiation.md)
A technique to attach derivative tracking to specific factors within an expression without recomputing the entire primal value, avoiding unnecessary extra computation in both primal and adjoint programs [11540-gradblock-with-automatic-differentiation.md].

### Differentiating Polymorphism via Dynamic VJP Generation
A technique to differentiate polymorphic method calls by dynamically generating Vector-Jacobian Product (VJP) functions for each method, wrapped in a `dr.CustomOp` [S1]. The process involves wrapping the polymorphic call (e.g., `out = obj.func(arg_1, arg_2, ..)`) into a dynamically generated `dr.CustomOp`. When `CustomOp.backward()` is invoked, a VJP per method is dynamically defined as a placeholder that calls the original function a second time with recursive AD usage. The JIT layer traces each possible target, running all VJP placeholder functions to materialize the differentiation. If a return value does not carry derivatives, the system devirtualizes the zero-valued derivative and propagates zero-valued input derivatives to simplify call targets using constant propagation [S1].

## Operating characteristics
- **Memory**: Requires storing intermediate steps and temporaries from the primal function for potential recomputation during the backward pass [S1].
- **Performance**: Dynamic VJP generation and JIT tracing add overhead during the first execution but optimize subsequent calls through constant propagation and devirtualization of zero-valued derivatives [S1].
- **Strengths**: Correctly handles implicit dependencies on global state, which is critical for differentiable rendering with scene parameters [S1].
- **Limitations**: Requires careful handling of atomic operations for implicit reads [S1].

## Background
The Custom Adjoint Interface is part of the Dr.Jit differentiable rendering framework, introduced in the paper "Dr.Jit: A Just-In-Time Compiler for Differentiable Rendering" [S1]. It addresses the limitations of standard AD systems that only consider explicit function arguments and return values, by extending the interface to track implicit inputs and outputs [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
