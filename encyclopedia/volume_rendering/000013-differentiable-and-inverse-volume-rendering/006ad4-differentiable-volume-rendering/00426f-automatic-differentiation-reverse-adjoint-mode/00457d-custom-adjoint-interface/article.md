# Custom Adjoint Interface

Also called: dr.CustomOp

## Overview
The Custom Adjoint Interface is a mechanism in differentiable programming systems that allows users to define custom forward and reverse-mode derivatives for functions, extending standard automatic differentiation (AD) by tracking implicit inputs and outputs (dependencies on global state) in addition to explicit function arguments and return values [S1]. It sets itself apart from standard AD interfaces by explicitly monitoring variable accesses during primal evaluation to automatically register implicit dependencies, ensuring correct gradient flow for operations involving global state such as scene parameters or texture lookups [S1].

## How it works
### Inputs and Outputs
Inputs include explicit function arguments (tensors/arrays) and implicit dependencies (global state accessed during execution) [S1]. Outputs include explicit return values and implicit updates to global state, such as scene parameter gradients [S1].

### Data Structures
The core data structure is the `dr.CustomOp` interface, which encapsulates a forward callback that computes the primal output from primal inputs [S1], a backward callback that computes input derivatives from output derivatives or vice versa [S1], and an implicit dependency tracker that monitors variable accesses during primal evaluation to automatically register implicit variable dependences with the AD system [S1].

### Steps
1. Registration: The user defines a `dr.CustomOp` subclass implementing the forward and backward callbacks [S1].
2. Primal Execution: When the function is called, the primal computation runs while the AD system monitors variable accesses to detect implicit dependencies on global state [S1].
3. Implicit Dependency Registration: Any variable accessed during the primal execution that is not an explicit argument is registered as an implicit input or output, ensuring the global flow of derivatives is correctly represented [S1].
4. Backward Invocation: When `dr.backward()` is called, the AD system traverses the computational tape and invokes the `backward()` callback upon encountering a `CustomOp` [S1].
5. Gradient Propagation: The `backward()` callback converts output derivatives into input derivatives, and for methods with implicit reads, the system issues atomic scatter-reductions to update scene parameter gradients [S1].

### Pseudocode
The following pseudocode illustrates the structure of a `dr.CustomOp` and its usage in Dr.Jit. It assumes a standard AD framework with gradient tracking and backward propagation operations [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MyCustomOp:
    def forward(self, x, y):
        result = x * y
        return result

    def backward(self, grad_out, x, y):
        grad_x = grad_out * y
        grad_y = grad_out * x
        return grad_x, grad_y

# Usage
x = [1.0, 2.0]
y = [3.0, 4.0]
op = MyCustomOp()
result = op.forward(x, y)

# Backpropagation
result_grad = [1.0, 1.0]
grad_x, grad_y = op.backward(result_grad, x, y)
```

### Formulas
Not given in the sources.

### Parameters
Not given in the sources.

### Edge Cases
Implicit reads of global state (e.g., scene parameters) require atomic scatter-reductions to update gradients during reverse-mode differentiation [S1]. Differentiating polymorphic method calls requires dynamic generation of Vector-Jacobian Product (VJP) functions, wrapped in a `dr.CustomOp` [S1]. If a return value does not carry derivatives, the system devirtualizes the zero-valued derivative and propagates zero-valued input derivatives to simplify call targets using constant propagation [S1]. Precision and boundary conditions are not discussed in the sources.

## Subtopics

### [GradBlock with Automatic Differentiation](11540-gradblock-with-automatic-differentiation.md)
GradBlock (also known as AD Tape Surgery) is a technique to attach derivative tracking to specific factors within an expression without recomputing the entire primal value, avoiding unnecessary extra computation in both primal and adjoint programs. It solves the problem of efficiently differentiating parts of a computation graph that were originally computed with detached or non-differentiable operations (e.g., GPU texture lookups or importance sampling strategies). It sets itself apart from naive substitution methods by combining the primal value of one variable with the AD trace of another, thereby bypassing redundant forward and backward passes.

### Differentiating Polymorphism via Dynamic VJP Generation
A technique to differentiate polymorphic method calls by dynamically generating Vector-Jacobian Product (VJP) functions for each method, wrapped in a `dr.CustomOp` [S1]. It sets itself apart by handling dynamic dispatch in differentiable code, allowing polymorphic calls to be differentiated without static knowledge of all possible targets [S1]. It is chosen when differentiating code with dynamic method resolution (e.g., `obj.func()` where `obj` is an array of instances) [S1].

#### Steps
1. Wrap the polymorphic method call (e.g., `out = obj.func(arg_1, arg_2, arg_3)`) into a dynamically generated `dr.CustomOp` [S1].
2. When `CustomOp.backward()` is invoked, dynamically define a VJP per method as a placeholder that calls the original function a second time with recursive AD usage [S1].
3. Issue a polymorphic method call to the newly defined VJP functions [S1].
4. The JIT layer traces each possible target, running all VJP placeholder functions to materialize the differentiation into ordinary traced computation [S1].
5. If a return value does not carry derivatives, devirtualize the zero-valued derivative and propagate zero-valued input derivatives to simplify call targets using constant propagation [S1].

#### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def func_vjp(grad_out, arg_1, arg_2, arg_3):
    arg_1.requires_grad = True
    arg_2.requires_grad = True
    arg_3.requires_grad = True
    result = func(arg_1, arg_2, arg_3)
    result.grad = grad_out
    result.backward()
    return arg_1.grad, arg_2.grad, arg_3.grad
```
This VJP function is dynamically generated for each method in the polymorphic call [S1].

## Operating characteristics
Time complexity depends on the complexity of the forward and backward callbacks, with dynamic VJP generation for polymorphism adding overhead during JIT tracing but simplifying execution via constant propagation for zero-valued derivatives [S1, G]. Memory complexity requires storing the computational graph and intermediate values for reverse-mode AD [G]. Strengths include correctly handling global state and implicit dependencies, which is crucial for differentiable rendering with scene parameters, and supporting polymorphism through dynamic VJP generation [S1]. Limitations require careful implementation of forward and backward callbacks to avoid incorrect gradient flow [G]. It is used when differentiating functions with global state dependencies or polymorphic calls that standard AD systems cannot handle efficiently [S1].

## Background
The Custom Adjoint Interface is part of the Dr.Jit differentiable rendering framework, introduced in the paper "Dr.Jit: A Just-In-Time Compiler for Differentiable Rendering" [S1]. It addresses the limitations of standard AD systems in handling implicit dependencies and polymorphism, which are common in rendering code [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
