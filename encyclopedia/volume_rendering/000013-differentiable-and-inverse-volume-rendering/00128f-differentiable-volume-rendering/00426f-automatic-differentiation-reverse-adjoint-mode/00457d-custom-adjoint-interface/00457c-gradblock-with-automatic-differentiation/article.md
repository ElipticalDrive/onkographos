# GradBlock with Automatic Differentiation

Also called: AD Tape Surgery, GradBlock

## Overview
GradBlock (also known as AD Tape Surgery) is a technique to attach derivative tracking to specific factors within an expression without recomputing the entire primal value, avoiding unnecessary extra computation in both primal and adjoint programs [S1]. It solves the problem of efficiently differentiating parts of a computation graph that were originally computed with detached or non-differentiable operations (e.g., GPU texture lookups or importance sampling strategies) [S1]. It sets itself apart from naive substitution methods by combining the primal value of one variable with the AD trace of another, thereby bypassing redundant forward and backward passes [S1].

## How it works
**Inputs and Outputs:**
- Input: Two variables, $a$ (primal value) and $b$ (differential value/trace).
- Output: A new variable that carries the primal value of $a$ but is linked to the AD tape via $b$.

**Data Structures:**
- The implementation relies on the underlying AD tape structure of the framework (e.g., Dr.Jit). The function `dr.replace_grad` manipulates the tape to replace the dependency of a node on its original parent with a dependency on $b$, while retaining the computed value of $a$ [S1].

**Steps:**
1. Identify the variable $a$ whose primal value is already computed and correct, but whose gradient flow needs to be redirected or attached to a differentiable context [S1].
2. Identify the variable $b$ that contains the desired AD trace or differential information [S1].
3. Call `dr.replace_grad(a, b)` to create a new variable $c$ [S1].
   - $c$ holds the value of $a$.
   - $c$ is attached to the AD tape via $b$.
4. Use $c$ in subsequent computations [S1].

**Formulas:**
The operation can be conceptually described as:
$$ c = \text{dr.replace\_grad}(a, b) $$
Where:
- $val(c) = val(a)$
- $\frac{\partial L}{\partial c}$ flows back through the tape associated with $b$ [S1].

In the context of rescaling gradients (e.g., for BSDF values), the sources show usage like:
$$ L_r \leftarrow L_r \times \text{dr.replace\_grad}(1, \frac{bsdf\_value\_diff}{bsdf\_value}) $$
Here, $bsdf\_value$ and $bsdf\_value\_diff$ have the same value, but $bsdf\_value\_diff$ is attached to the AD graph [S1].

**Parameters:**
- $a$: The primal value variable.
- $b$: The variable with the desired AD trace.

**Pitfalls:**
- Naively changing a factor in the expression (e.g., `Lr *= bsdf_value_diff / bsdf_value`) introduces unnecessary extra computation in both primal and adjoint programs [S1].
- Precision issues may arise if $bsdf\_value$ is near zero when dividing [G].
- The method assumes the framework supports tape manipulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def replace_grad_example():
    # Example from [S1]: Attaching AD trace to a BSDF value
    # bsdf_value is the primal value
    # bsdf_value_diff is the same value but attached to AD tape
    bsdf_value = ... # Computed primal value
    bsdf_value_diff = ... # Same value, differentiable
    
    # Naive approach (inefficient):
    # Lr *= bsdf_value_diff / bsdf_value 
    
    # Efficient approach using GradBlock:
    Lr = Lr * dr.replace_grad(1, bsdf_value_diff / bsdf_value)
    
    # Example from [S1]: Turning GPU texture lookups into differentiable operations
    # tex_lookup is a non-differentiable GPU texture lookup
    # soft_lookup is a differentiable software-based lookup
    tex_val = gpu_texture_lookup(...) # Non-differentiable
    soft_val = soft_texture_lookup(...) # Differentiable
    
    # Combine primal value of tex_val with AD trace of soft_val
    diff_tex_val = dr.replace_grad(tex_val, soft_val)
    
    # Now diff_tex_val can be used in differentiable computations
    return diff_tex_val
```

## Subtopics

### [Isolation Boundaries](11535-isolation-boundaries.md)
A mechanism to restrain the process of derivative propagation. Only edges within and just across the isolation boundary may be traversed; others are postponed until the scope is destructed. Used to ensure correct generation of a differential megakernel embedded within a larger differential calculation.

## Operating characteristics
- **Time Complexity:** Avoids recomputing the primal value, thus saving forward pass time [S1]. The cost is the overhead of tape manipulation [S1].
- **Memory Complexity:** No significant additional memory overhead compared to standard AD, as it reuses existing tape structures [S1].
- **Performance:** In benchmarks, using `dr.replace_grad` reduced time overhead compared to naive substitution [S1]. For example, in the "Staircase" scene, it contributed to a 0.71x speedup in kernel execution time compared to a baseline [S1].
- **Strengths:** Efficiently enables differentiation of detached operations without redundant computation [S1].
- **Limitations:** Requires framework support for tape surgery [S1]. Not applicable to operations that cannot be represented in the AD tape [G].

## Background
GradBlock was introduced in the Dr.Jit paper [S1]. It is used in differentiable rendering to handle cases where importance sampling or texture lookups are not naturally differentiable [S1]. The technique allows for the generation of nonzero derivatives in scenarios like reflected radiance calculations with detached BSDF sampling strategies [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
[S2] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
