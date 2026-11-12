# Forward-Mode Automatic Differentiation

Also called: Forward Differentiation, Forward mode differentiation, Tangent Mode AD, Tape-based Automatic Differentiation, Tracing (Automatic Differentiation), Wengert tape, computation graph

## Overview
Forward-mode automatic differentiation (AD) computes the derivatives of a function's output with respect to its inputs by propagating derivative information simultaneously with the primal values through the computational graph [S1, S2]. It solves the problem of obtaining exact gradients for differentiable rendering by traversing the graph from inputs to outputs, making it efficient when the input dimension is low and the output dimension is high [S1, S2]. Unlike reverse-mode AD, which requires a backward pass and is better suited for high-dimensional inputs (like volumetric fields), forward-mode requires a separate pass for each input parameter, making it less suitable for optimizing high-dimensional parameters but ideal for visualizing the effect of specific scene parameters (e.g., camera position or single material properties) [S6, S7].

## How it works
**Inputs and Outputs:**
- **Inputs:** A differentiable function $f: \mathbb{R}^d \to \mathbb{R}^m$ (e.g., a rendering function), represented as a sequence of elementary operations.
- **Outputs:** The primal values of the function and the Jacobian-vector products (derivatives of outputs with respect to specific input parameters).

**Data Structures:**
The core data structure is the **forward variable** (often called `fvar` or dual number), which wraps a primal value and its derivatives [S2][S3, unofficial].

1.  **Dual Number / Forward Variable (`fvar`):**
    -   Stores a scalar value `value` and a derivative vector `derivatives` [S2].
    -   Layout:
        ```python
        struct fvar<T, p>:
            T value      # The primal value
            T derivatives[p] # Derivatives with respect to p parameters
        ```
    -   For a single parameter $w$, the derivative is a scalar. For $p$ parameters, `derivatives` is an array of size $p$ [S2].
    -   In some implementations (e.g., for scalar fields), the gradient is a vector $\mathbb{R}^n$ [S3, unofficial].

2.  **Computation Graph / Tape:**
    -   A record of all differentiable arithmetic operations performed during the forward pass [S4].
    -   Nodes represent variables; edges represent dependencies and derivative weights [S4].
    -   Control flow (loops, branches) is unrolled or traced into the graph [S4].

**Numbered Steps:**

1.  **Initialization:**
    -   Initialize constant variables with zero derivative: $\tilde{x}_{const} = \langle x_{const}, 0 \rangle$ [S2].
    -   Initialize parameters of interest with unit derivative: $\tilde{w} = \langle w, 1 \rangle$ [S2].
    -   If computing derivatives for multiple parameters $w_1, \dots, w_p$, initialize $\tilde{w}_i$ such that its $i$-th derivative is 1 and others are 0 (identity matrix initialization) [S1, S2].
    -   Example for two inputs $x_0, x_1$:
        -   $\tilde{x}_0 = \langle x_0, [1, 0] \rangle$
        -   $\tilde{x}_1 = \langle x_1, [0, 1] \rangle$ [S1]

2.  **Forward Propagation (Operator Overloading):**
    -   Execute the primal computation using `fvar` types instead of standard scalars [S2].
    -   Operator overloading ensures that arithmetic operations apply the chain rule automatically [S2].
    -   For each operation $c = f(a, b)$, compute:
        $$ \tilde{c} = \tilde{f}(\tilde{a}, \tilde{b}) = \left\langle f(a, b), \frac{\partial f}{\partial a} \frac{da}{dw} + \frac{\partial f}{\partial b} \frac{db}{dw} \right\rangle $$ [S2]

3.  **Elementary Operation Rules:**
    -   **Addition:** $\tilde{c} = \tilde{a} + \tilde{b}$
        -   $c.value = a.value + b.value$
        -   $c.derivatives[i] = a.derivatives[i] + b.derivatives[i]$ [S3, unofficial]
    -   **Multiplication:** $\tilde{c} = \tilde{a} \cdot \tilde{b}$
        -   $c.value = a.value \cdot b.value$
        -   $c.derivatives[i] = a.value \cdot b.derivatives[i] + b.value \cdot a.derivatives[i]$ [S2][S3, unofficial]
    -   **Division:** $\tilde{c} = \tilde{a} / \tilde{b}$
        -   $c.value = a.value / b.value$
        -   $c.derivatives[i] = \frac{a.derivatives[i] \cdot b.value - a.value \cdot b.derivatives[i]}{b.value^2}$ [S3, unofficial]
    -   **Exponential:** $\tilde{c} = \exp(\tilde{a})$
        -   $c.value = \exp(a.value)$
        -   $c.derivatives[i] = c.value \cdot a.derivatives[i]$ [S1]
    -   **Square Root:** $\tilde{c} = \sqrt{\tilde{a}}$
        -   $c.value = \sqrt{a.value}$
        -   $c.derivatives[i] = \frac{a.derivatives[i]}{2 \cdot c.value}$ [S3, unofficial]
    -   **Sine:** $\tilde{c} = \sin(\tilde{a})$
        -   $c.value = \sin(a.value)$
        -   $c.derivatives[i] = a.derivatives[i] \cdot \cos(a.value)$ [S3, unofficial]
    -   **Cosine:** $\tilde{c} = \cos(\tilde{a})$
        -   $c.value = \cos(a.value)$
        -   $c.derivatives[i] = -a.derivatives[i] \cdot \sin(a.value)$ [S3, unofficial]
    -   **Absolute Value:** $\tilde{c} = |\tilde{a}|$
        -   If $a.value < 0$, $c.derivatives[i] = -a.derivatives[i]$; else $c.derivatives[i] = a.derivatives[i]$ [S3, unofficial]

4.  **Latent Variable Handling (Advanced):**
    -   If a scene parameter depends on another differentiable latent variable, gradients must be propagated from the latent variable to the parameter before rendering [S6, S7].
    -   Use `dr.enable_grad(latent_var)` to mark the latent variable as differentiable [S6].
    -   Propagate gradients using `dr.forward(latent_var, dr.ADFlag.ClearEdges)` to ensure intermediate parameters retain their gradients and are not zeroed out [S6, S7].
    -   Render the scene to record the computation graph [S6].
    -   Propagate gradients to the output image using `dr.forward_to(output_image)` to accumulate gradients from all sources without overwriting intermediate values [S6, S7].

5.  **Result Extraction:**
    -   Retrieve the gradient of the output (e.g., image) using `dr.grad(output)` [S6, S7].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Definition of Forward Variable
class fvar:
    def __init__(self, value, derivatives):
        self.value = value
        self.derivatives = derivatives # List or array of size p

    def __add__(self, other):
        # Element-wise addition of derivatives
        new_derivs = [self.derivatives[i] + other.derivatives[i] for i in range(len(self.derivatives))]
        return fvar(self.value + other.value, new_derivs)

    def __mul__(self, other):
        # Product rule for derivatives
        new_derivs = [self.value * other.derivatives[i] + other.value * self.derivatives[i] 
                      for i in range(len(self.derivatives))]
        return fvar(self.value * other.value, new_derivs)

    def __truediv__(self, other):
        # Quotient rule for derivatives
        denom_sq = other.value * other.value
        new_derivs = [(self.derivatives[i] * other.value - self.value * other.derivatives[i]) / denom_sq
                      for i in range(len(self.derivatives))]
        return fvar(self.value / other.value, new_derivs)

    def exp(self):
        val = exp(self.value)
        new_derivs = [val * self.derivatives[i] for i in range(len(self.derivatives))]
        return fvar(val, new_derivs)

    def sin(self):
        val = sin(self.value)
        new_derivs = [self.derivatives[i] * cos(self.value) for i in range(len(self.derivatives))]
        return fvar(val, new_derivs)

# Example: Differentiating f(x, y) = x * sin(y) with respect to x and y
def forward_mode_example():
    # Initialize inputs: x with deriv [1, 0], y with deriv [0, 1]
    x = fvar(2.0, [1.0, 0.0])
    y = fvar(3.0, [0.0, 1.0])

    # Compute primal and derivative
    # Step 1: sin(y)
    sin_y = y.sin()
    
    # Step 2: x * sin(y)
    result = x * sin_y
    
    return result.value, result.derivatives

# result.value = 2.0 * sin(3.0)
# result.derivatives[0] (dx) = 1.0 * sin(3.0) + 2.0 * 0.0 = sin(3.0)
# result.derivatives[1] (dy) = 2.0 * cos(3.0) + 0.0 * sin(3.0) = 2.0 * cos(3.0)
```

**Parameters:**
-   $p$: Number of input parameters. The derivative array size is $p$ [S2].
-   Typical values: $p=1$ for single parameter sensitivity; $p=2$ for camera viewpoint (longitude, latitude) [S2].

**Edge Cases and Pitfalls:**
-   **High Dimensionality:** Forward mode becomes unfeasible for high-dimensional inputs (e.g., 3D volumetric fields, large Transfer Functions) because the derivative array size $p$ grows linearly, requiring significant memory and computation [S2][S5, unofficial].
-   **Visualization Clipping:** Gradient values are not bounded to $[0, 1]$. Directly displaying them as RGB images will clip or distort results. Use colormaps (e.g., coolwarm) and clamp display ranges to $[-v_{lim}, v_{lim}]$ where $v_{lim}$ is the max absolute gradient value [S6, S7].
-   **Latent Variables:** If gradients of latent variables are not propagated to scene parameters before rendering, the computational graph may be incorrectly truncated, leading to incorrect gradients [S7].
-   **Control Flow:** Differentiating long-running loops can be challenging due to runtime overheads and the need to unroll traces [S4].

## Subtopics

### [Dual Number Automatic Differentiation for Scalar Fields](5494-dual-number-automatic-differentiation-for-scalar-fields.md)
Computes exact gradients of scalar fields using dual numbers, primarily for implicit surface normals. Chosen when avoiding finite difference errors is critical.

### [Tracing-based Automatic Differentiation with Dynamic Compilation](11558-tracing-based-automatic-differentiation-with-dynamic-compila.md)
Combines Wengert tape tracing with JIT compilation to optimize derivative code. Chosen to reduce runtime overhead of pure tracing AD while maintaining flexibility.

## Operating characteristics
-   **Time Complexity:** $O(d \cdot T)$, where $d$ is the number of input variables and $T$ is the time complexity of the original algorithm [S1]. Scales linearly with the number of parameters $p$ [S2][S5, unofficial].
-   **Memory Complexity:** $O(n \cdot p)$, where $n$ is the number of operations or intermediate results (e.g., image pixels) and $p$ is the number of parameters [S2][S5, unofficial].
-   **Performance:** Efficient for low-dimensional inputs (e.g., camera parameters) [S2]. Inefficient for high-dimensional inputs (e.g., volume densities) [S2][S5, unofficial].
-   **Strengths:** Exact derivatives; simpler implementation than source transformation; easier to integrate into existing code via operator overloading [S1].
-   **Limitations:** Poor scalability with input dimension; requires separate pass per parameter for optimization [S6].
-   **When to Use:** Visualizing parameter effects; optimizing few parameters (e.g., camera pose); low-dimensional scalar fields.
-   **When Not to Use:** Optimizing high-dimensional parameters (e.g., volume reconstruction, large TFs); large-scale inverse rendering where reverse-mode is preferred [S2, S6].

## Background
Forward-mode automatic differentiation was introduced as a method to compute derivatives by recursively applying the chain rule to elementary operations [S1]. It is based on the algebraic properties of dual numbers [S3, unofficial]. The method traverses the computational graph from inputs to outputs, computing derivatives of intermediate nodes with respect to all input variables [S1]. It is widely used in differentiable rendering for sensitivity analysis and is implemented in libraries like PyTorch (via tracing) and Dr.Jit [S1, S4]. In volume rendering, it is used for viewpoint selection and visualizing the impact of scene parameters [S2, S6].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] Differentiable Direct Volume Rendering, https://arxiv.org/abs/2107.12672v1, fetched 2026-10-26
[S3] planaria/dmc, https://github.com/planaria/dmc, fetched 2026-10-26 (unofficial)
[S4] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
[S5] (PDF) Differentiable Direct Volume Rendering - ResearchGate, https://www.researchgate.net/publication/353510271_Differentiable_Direct_Volume_Rendering, fetched 2026-10-30 (unofficial)
[S6] Forward inverse rendering, https://mitsuba.readthedocs.io/en/stable/src/inverse_rendering/forward_inverse_rendering.html, fetched 2026-10-29
[S7] Forward inverse rendering, https://mitsuba.readthedocs.io/en/latest/src/inverse_rendering/forward_inverse_rendering.html, fetched 2026-10-29
