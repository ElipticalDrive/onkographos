# Custom Adjoint Interface

Also called: dr.CustomOp

## Overview
The Custom Adjoint Interface is a mechanism in differentiable programming systems that allows users to define custom forward and reverse-mode derivatives for functions, extending standard automatic differentiation (AD) by tracking implicit inputs and outputs (dependencies on global state) in addition to explicit function arguments and return values [S1]. It sets itself apart from standard AD interfaces by explicitly monitoring variable accesses during primal evaluation to automatically register implicit dependencies, ensuring correct gradient flow for operations involving global state such as scene parameters or texture lookups [S1].

## How it works
The interface operates by intercepting function calls within the AD computational graph. When a user defines a custom operation, the system registers it as a `CustomOp`. During the forward pass, the primal computation is executed. During the backward pass (reverse mode), the AD system traverses the tape and invokes the `CustomOp`'s callbacks to propagate gradients.

### Inputs and Outputs
*   **Inputs:** Explicit function arguments (tensors/arrays) and implicit dependencies (global state accessed during execution).
*   **Outputs:** Explicit return values and implicit updates to global state (e.g., scene parameter gradients).

### Data Structures
The core data structure is the `CustomOp` object, which encapsulates:
1.  **Forward Callback:** A function or method that computes the primal output from primal inputs.
2.  **Backward Callback:** A function or method that computes input gradients from output gradients.
3.  **Implicit Dependency Tracker:** A mechanism to monitor variable accesses during the forward pass to identify implicit inputs/outputs [S1].

### Steps
1.  **Registration:** The user defines a `CustomOp` with forward and backward logic. The system registers this operation.
2.  **Primal Execution:** When the function is called, the primal computation runs. The AD system monitors variable accesses to detect implicit dependencies on global state [S1].
3.  **Implicit Dependency Registration:** Any variable accessed during the primal execution that is not an explicit argument is registered as an implicit input or output. This ensures the global flow of derivatives is correctly represented [S1].
4.  **Backward Invocation:** When `dr.backward()` is called, the AD system traverses the tape. Upon encountering a `CustomOp`, it invokes the `backward()` callback.
5.  **Gradient Propagation:** The `backward()` callback converts output derivatives into input derivatives. For methods with implicit reads (e.g., accessing scene parameters), the system issues atomic scatter-reductions to update the gradients of those scene parameters [S1].

### Pseudocode
The following pseudocode illustrates the structure of a `CustomOp` and its usage in Dr.Jit. It assumes a standard AD framework with `dr.enable_grad`, `dr.set_grad`, `dr.backward`, and `dr.grad` operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Define a custom operation class
class MyCustomOp(dr.CustomOp):
    def forward(self, x, y):
        # Primal computation
        # Implicit dependencies (e.g., global texture) are accessed here
        # The AD system monitors these accesses automatically
        result = x * y + global_texture_lookup(x)
        return result

    def backward(self, grad_out):
        # Reverse-mode derivative
        # grad_out is the gradient of the loss with respect to result
        # Compute gradients for x and y
        grad_x = grad_out * y
        grad_y = grad_out * x
        # Implicit dependencies (global_texture_lookup) gradients are handled
        # by the system via atomic scatter-reductions if they are scene parameters
        return grad_x, grad_y

# Usage
x = dr.Variable(...)
y = dr.Variable(...)
# Register the custom op
op = MyCustomOp()
result = op.forward(x, y)

# Backpropagation
dr.set_grad(result, 1.0) # Set gradient of the result
dr.backward(result)       # Traverse tape and invoke backward()
grad_x = dr.grad(x)
grad_y = dr.grad(y)
```

### Parameters and Edge Cases
*   **Implicit Dependencies:** Must be correctly identified. Failure to track implicit reads/writes leads to incorrect gradients for global state [S1].
*   **Atomic Scatter-Reductions:** Used for updating scene parameter gradients from implicit reads. This is the main way scene parameter gradients are generated [S1].
*   **Polymorphism:** Differentiating polymorphic method calls requires dynamic generation of Vector-Jacobian Product (VJP) functions, wrapped in a `dr.CustomOp` [S1].

## Subtopics

### [GradBlock with Automatic Differentiation](11540-gradblock-with-automatic-differentiation.md)
GradBlock (also known as AD Tape Surgery) is a technique to attach derivative tracking to specific factors within an expression without recomputing the entire primal value, avoiding unnecessary extra computation in both primal and adjoint programs. It solves the problem of efficiently differentiating parts of a computation graph that were originally computed with detached or non-differentiable operations (e.g., GPU texture lookups or importance sampling strategies). It sets itself apart from naive substitution methods by combining the primal value of one variable with the AD trace of another, thereby bypassing redundant forward and backward passes.

### Differentiating Polymorphism via Dynamic VJP Generation
A technique to differentiate polymorphic method calls by dynamically generating Vector-Jacobian Product (VJP) functions for each method, wrapped in a `dr.CustomOp` [S1]. It sets itself apart by handling dynamic dispatch in differentiable code, allowing polymorphic calls to be differentiated without static knowledge of all possible targets. It is chosen when differentiating code with dynamic method resolution (e.g., `obj.func()` where `obj` is an array of instances) [S1].

#### Steps
1.  Wrap the polymorphic method call (e.g., `out = obj.func(arg_1, arg_2, ..)`) into a dynamically generated `dr.CustomOp` [S1].
2.  When `CustomOp.backward()` is invoked, dynamically define a VJP per method as a placeholder that calls the original function a second time with recursive AD usage [S1].
3.  Issue a polymorphic method call to the newly defined VJP functions [S1].
4.  The JIT layer traces each possible target, running all VJP placeholder functions to materialize the differentiation into ordinary traced computation [S1].
5.  If a return value does not carry derivatives, devirtualize the zero-valued derivative and propagate zero-valued input derivatives to simplify call targets using constant propagation [S1].

#### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def func_vjp(grad_out, arg_1, arg_2, ..):
    dr.enable_grad(arg_1, arg_2, ..)
    result = func(arg_1, arg_2, ..)
    dr.set_grad(result, grad_out)
    dr.backward(result)
    return dr.grad(arg_1), dr.grad(arg_2), ..
```
This VJP function is dynamically generated for each method in the polymorphic call [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the complexity of the forward and backward callbacks. Dynamic VJP generation for polymorphism adds overhead during JIT tracing but simplifies execution via constant propagation for zero-valued derivatives [S1].
*   **Memory Complexity:** Requires storing the computational graph and intermediate values for reverse-mode AD. Implicit dependency tracking adds minimal overhead [S1].
*   **Strengths:** Correctly handles global state and implicit dependencies, which is crucial for differentiable rendering with scene parameters. Supports polymorphism through dynamic VJP generation [S1].
*   **Limitations:** Requires careful implementation of forward and backward callbacks. Implicit dependency tracking must be robust to avoid missing gradients [S1].
*   **When to Use:** When differentiating functions with global state dependencies or polymorphic calls that standard AD systems cannot handle efficiently [S1].

## Background
The Custom Adjoint Interface is part of the Dr.Jit differentiable rendering framework, introduced in the paper "Dr.Jit: A Just-In-Time Compiler for Differentiable Rendering" [S1]. It addresses the limitations of standard AD systems in handling implicit dependencies and polymorphism, which are common in rendering code [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
