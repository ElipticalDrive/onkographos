# Weighted Sum of Phase Functions
Also called: Composite Phase Function (Weighted Sum), Weighted Sum Phase Function

## Overview
The Weighted Sum of Phase Functions is a method for modeling complex scattering distributions that cannot be accurately described by a single asymmetry parameter. It generalizes the Henyey-Greenstein phase function by combining multiple phase function instances with different parameters into a single normalized model. It sets itself apart from its parent, the [Multiple Octaves Phase Function](4939-multiple-octaves-phase-function.md), by providing a general framework for any number of lobes, whereas the parent specifically describes a two-lobe configuration using Henyey-Greenstein components. It is distinct from the [Dual-Lobe Henyey-Greenstein Phase Function](5231-dual-lobe-henyey-greenstein-phase-function.md) in that it is not restricted to exactly two lobes or a specific lobe structure.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ base phase functions $p_i$, each defined by its own parameters (e.g., asymmetry parameter $g_i$ for Henyey-Greenstein), and a set of weights $w_i$ for each phase function.
*   **Outputs:** A composite phase function value $p(\omega_o, \omega'_o)$ for given outgoing direction $\omega_o$ and incoming direction $\omega'_o$.

**Data Structures:**
*   **Phase Function List:** An array or list of $n$ phase function objects. Each object contains the parameters required by its specific type (e.g., $g_i$ for Henyey-Greenstein).
*   **Weights Array:** A vector of $n$ floating-point numbers $w_i$.

**Steps:**
1.  Initialize the total phase value $P_{total} = 0$.
2.  For each phase function index $i$ from $1$ to $n$:
    a. Evaluate the base phase function $p_i(\omega_o, \omega'_o)$ using its specific parameters.
    b. Multiply the result by the corresponding weight $w_i$.
    c. Add the result to $P_{total}$.
3.  Return $P_{total}$.

**Formula:**
The composite phase function is defined as:
$$ p(\omega_o, \omega'_o) = \sum_{i=1}^n w_i p_i(\omega_o \rightarrow \omega'_o) $$
where:
*   $p_i(\omega_o \rightarrow \omega'_o)$ is the value of the $i$-th base phase function for directions $\omega_o$ and $\omega'_o$.
*   $w_i$ is the weight for the $i$-th phase function.
*   The weights must satisfy the normalization constraint:
    $$ \sum_{i=1}^n w_i = 1 $$

**Parameters:**
*   $n$: The number of constituent phase functions. Not given in the sources.
*   $w_i$: Weights for each phase function. Must sum to 1.

**Edge Cases and Pitfalls:**
*   **Normalization:** The weights $w_i$ must sum to one to maintain normalization of the phase function. If they do not sum to one, the resulting phase function will not be properly normalized, leading to incorrect energy conservation in rendering [S1].
*   **Implementation:** This generalization is not provided in pbrt but is noted as easy to add [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def weighted_sum_phase_function(outgoing_dir, incoming_dir, phase_functions, weights):
    """
    Calculate the weighted sum of phase functions.
    
    Args:
        outgoing_dir: Vector representing the outgoing direction omega_o.
        incoming_dir: Vector representing the incoming direction omega'_o.
        phase_functions: List of phase function objects. Each object must have
                         an evaluate(outgoing_dir, incoming_dir) method.
        weights: List of float weights corresponding to each phase function.
        
    Returns:
        The weighted sum of the phase function values.
    """
    total_value = 0.0
    for i in range(len(phase_functions)):
        # Evaluate the i-th phase function
        p_i = phase_functions[i].evaluate(outgoing_dir, incoming_dir)
        # Multiply by weight and add to total
        total_value += weights[i] * p_i
        
    return total_value
```

## Subtopics

### Total Phase Function
The Total Phase Function is a specific application of the weighted sum concept where the weights are determined by the contribution of each particle field to the total scattering coefficient. It is calculated by weighting each individual particle phase function $P^{(i)}(\Omega)$ by the product of its volume fraction $\alpha^{(i)}$ and its extinction coefficient $\beta_{ex}^{(i)}(\mathbf{s})$. The formula is:
$$ P(\Omega, \mathbf{s}) = \frac{\sum_{\text{all fields } i} \alpha^{(i)} \beta_{ex}^{(i)}(\mathbf{s}) P^{(i)}(\Omega)}{\sum_{\text{all fields } i} \alpha^{(i)} \beta_{ex}^{(i)}(\mathbf{s})} $$
This method is used in rendering atmospheric data with multiple particle fields to capture realistic illumination effects [S2].

## Operating characteristics
*   **Time Complexity:** $O(n)$ per evaluation, where $n$ is the number of constituent phase functions. Each constituent phase function must be evaluated independently.
*   **Memory Complexity:** $O(n)$ to store the parameters and weights for the $n$ phase functions.
*   **Strengths:** Allows modeling of complex scattering patterns (e.g., multi-lobe distributions) that a single Henyey-Greenstein function cannot capture. It is flexible and can incorporate any base phase function.
*   **Limitations:** More computationally expensive than a single phase function due to the summation. Requires careful tuning of weights and parameters for each constituent function.
*   **When to use:** When the scattering behavior of a medium is complex and cannot be approximated well by a single asymmetry parameter.
*   **When not to use:** When performance is critical and a simpler phase function provides sufficient accuracy.

## Background
The concept of modeling complex phase functions as a weighted sum of simpler phase functions (like Henyey-Greenstein) is discussed in the context of volume scattering in computer graphics. It is noted as a generalization that is not natively provided in the pbrt renderer but is considered easy to implement [S1]. The specific application of weighting phase functions by their contribution to the total scattering coefficient for atmospheric rendering is described in the context of efficient rendering of atmospheric phenomena [S2].

## Sources
[S1] 11.2 Phase Functions - pbr-book.org, https://pbr-book.org/3ed-2018/Volume_Scattering/Phase_Functions, fetched 2026-10-27
[S2] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
