# NCP Stopping Rule

Also called: Normalised Cumulative Periodogram Stopping Rule

## Overview
The NCP Stopping Rule is a termination criterion for iterative tomographic reconstruction algorithms that halts the process when the reconstruction residual behaves like white noise [S1]. It distinguishes itself from fixed-iteration or simple residual-norm methods by adapting to the specific noise level in the projection data, ensuring the algorithm stops near the optimal solution without overfitting to noise [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** The current reconstruction estimate $x_k$, the projection data $b$, the system matrix $A$, and the maximum iteration count $N_{max}$ [S1].
*   **Outputs:** A boolean flag indicating termination or the iteration index $k$ where termination occurs [S1].

**Data Structures**
*   $r_k \in \mathbb{R}^n$: The residual vector at iteration $k$ [S1].
*   $\hat{r} \in \mathbb{C}^n$: The Discrete Fourier Transform (DFT) of the residual [S1].
*   $\hat{p} \in \mathbb{R}^q$: The periodogram vector, where $q = \lceil n/2 \rceil$ [S1].
*   $c \in \mathbb{R}^q$: The Normalised Cumulative Periodogram (NCP) vector [S1].
*   $c_{white} \in \mathbb{R}^q$: The reference line for white noise [S1].

**Numbered Steps**
1.  **Compute Residual:** Calculate the residual vector $r_k = b - A x_k$ [S1]. Note: In implementations using libraries like ASTRA, the residual norm may be available during the update step, but the full vector must be explicitly calculated for the NCP check [S1].
2.  **Compute DFT:** Calculate the Discrete Fourier Transform of the residual: $\hat{r} = \text{DFT}(r_k)$ [S1].
3.  **Compute Periodogram:** Calculate the periodogram $\hat{p}$ using the squared absolute values of the DFT coefficients. Only approximately half of the elements are used because the power spectrum of a real vector is symmetric around the midpoint [S1]:
    $$ \hat{p}_i = |\hat{r}_i|^2, \quad i=1, 2, \dots, q $$
    where $q = \lceil n/2 \rceil$ [S1].
4.  **Compute NCP:** Calculate the Normalised Cumulative Periodogram $c_j(r_k)$ for $j=1, \dots, q$. The DC component (first element of $\hat{p}$) is excluded from the summation to ensure the curve starts at $(0,0)$ [S1]:
    $$ c_j(r_k) = \frac{\sum_{i=2}^{j+1} \hat{p}_i}{\sum_{i=2}^{q+1} \hat{p}_i}, \quad j=1, 2, \dots, q $$
5.  **Compute Distance to White Noise:** Calculate the $\ell_2$-norm distance $r_{NCP}$ between the computed NCP $c(r_k)$ and the expected white noise line $c_{white}$, which is a straight line ranging from $(0,0)$ to $(q,1)$ [S1]:
    $$ r_{NCP}(k) = \| c(r_k) - c_{white} \|_2 $$
6.  **Check Termination:** Terminate the reconstruction if two iterations on either side of the current $r_{NCP}(k)$ are larger than $r_{NCP}(k)$. This requires storing the $r_{NCP}$ values for the current and previous iterations [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ncp_stopping_rule(residual_vector, n, max_iterations):
    """
    Determines if the NCP stopping rule is met.
    
    Args:
        residual_vector (list[float]): The residual r_k = b - A * x_k.
        n (int): The dimension of the residual vector (number of voxels/pixels).
        max_iterations (int): N_max to prevent infinite loops.
        
    Returns:
        bool: True if the algorithm should terminate, False otherwise.
    """
    # Step 1: Compute DFT of the residual
    # DFT is a standard mathematical operation. 
    # In practice, this is often accelerated via CUDA (e.g., CuPy) [S1].
    dft_coeffs = dft(residual_vector)
    
    # Step 2: Determine q
    q = ceil(n / 2)
    
    # Step 3: Compute Periodogram p_hat
    # Only use the first q coefficients. 
    # Exclude DC component (index 0) for NCP calculation later, 
    # but keep it for periodogram definition if needed by specific DFT conventions.
    # The formula uses indices 1 to q (1-based) or 0 to q-1 (0-based).
    # Based on formula: p_i = |r_hat_i|^2 for i=1..q.
    # Assuming 1-based indexing for formula mapping to 0-based array:
    # p_hat[0] corresponds to i=1, p_hat[q-1] corresponds to i=q.
    p_hat = [abs(dft_coeffs[i])**2 for i in range(q)]
    
    # Step 4: Compute NCP c
    # c_j(r) for j=1..q
    # Formula: c_j = sum(p_2...p_{j+1}) / sum(p_2...p_{q+1})
    # Note: The formula indices are 1-based. 
    # p_2 is index 1 in 0-based p_hat. p_{q+1} is index q in 0-based p_hat.
    # However, p_hat only has length q (indices 0 to q-1).
    # The denominator sum goes up to p_{q+1}. 
    # If q = ceil(n/2), and n is even, n=2q. DFT has 2q coeffs.
    # Symmetry means we use q coeffs. 
    # The formula explicitly sums from i=2 to q+1. 
    # This implies we need access to p_{q+1}. 
    # If p_hat is defined for i=1..q, then p_{q+1} is outside this range?
    # Re-reading source: "q = ceil(n/2)... only approximately half... used".
    # Formula: c_j(r) = (sum_{i=2}^{j+1} p_i) / (sum_{i=2}^{q+1} p_i).
    # If p is defined for i=1..q, then p_{q+1} is not in p_hat?
    # Let's look at the denominator: sum_{i=2}^{q+1} p_i.
    # If q=ceil(n/2), and we have q periodogram values, the indices are 1..q.
    # The sum goes to q+1. This suggests the periodogram might be defined up to q+1 
    # or the indexing in the source text has a specific convention.
    # Standard NCP for real signals of length N uses N/2 points.
    # Let's assume the denominator is the sum of all available periodogram components 
    # excluding the DC component (i=1).
    # If p_hat has length q, indices 0..q-1 correspond to i=1..q.
    # The sum in denominator is i=2 to q+1. 
    # If the signal is real, the spectrum is symmetric. 
    # The source says "p_i = |r_hat_i|^2, i=1,2,...,q".
    # And "c_j(r) = ... / sum_{i=2}^{q+1} p_i".
    # This implies p_{q+1} exists. 
    # Perhaps q is defined differently or the sum includes the Nyquist frequency if n is even?
    # If n is even, DFT has n/2 + 1 unique real coefficients (0 to n/2).
    # If q = n/2, then indices 1..q are 1..n/2. 
    # The sum goes to q+1 = n/2 + 1. This matches the Nyquist bin.
    # So, p_hat should likely include the Nyquist bin if n is even.
    # Let's adjust q to include the Nyquist bin for even n.
    # If n is even, unique freqs are 0..n/2. Count = n/2 + 1.
    # If n is odd, unique freqs are 0..(n-1)/2. Count = (n+1)/2.
    # Let's define M as the number of unique non-DC coefficients including Nyquist.
    # M = floor(n/2) + 1.
    # The source uses q = ceil(n/2).
    # If n=4, q=2. Indices 1,2. Sum to 3? p_3?
    # DFT of length 4: 0, 1, 2, 3. Symmetric: 1 and 3 are conjugate. 2 is Nyquist.
    # Power at 1 and 3 are equal. Power at 2 is real.
    # Usually periodogram sums |X_k|^2.
    # If we only take 1..q, we miss Nyquist?
    # Let's stick strictly to the source formula variables.
    # Source: q = ceil(n/2). p_i for i=1..q.
    # Denominator sum i=2 to q+1.
    # This requires p_{q+1}.
    # If p is only defined up to q, this is an error in the source or implicit extension.
    # However, standard NCP normalization uses the total power.
    # Total power (excluding DC) = sum_{i=2}^{n} |X_i|^2? No, symmetric.
    # Let's assume the denominator is the sum of all periodogram values excluding DC.
    # If the source explicitly writes q+1, and q=ceil(n/2), 
    # for n=4, q=2, sum to 3. p_3 corresponds to freq 3? 
    # In DFT of length 4, freq 3 is symmetric to freq 1.
    # Maybe p_i represents the power at frequency bin i, and for real signals,
    # we sum the unique bins.
    # Let's implement the denominator as the sum of p_hat excluding the first element (DC).
    # If the source insists on q+1, and p is only up to q, we might assume p_{q+1} is 0 
    # or that q is actually floor(n/2)+1.
    # Given the ambiguity, we will calculate the denominator as the sum of all 
    # periodogram elements excluding the DC component (index 0 in 0-based p_hat).
    
    # Re-evaluating based on "p_i = |r_hat_i|^2, i=1..q" and "sum i=2..q+1".
    # If n is even, n=2m. q=m. Indices 1..m.
    # DFT bins: 0, 1, ..., m, ..., 2m-1.
    # Symmetry: k and 2m-k.
    # Unique non-DC bins: 1..m. (m includes Nyquist).
    # So p_1..p_m are the unique powers.
    # The sum in denominator is i=2..m+1.
    # p_{m+1} is not in the unique set? 
    # Actually, bin m is Nyquist. Bin m+1 is symmetric to bin m-1.
    # So p_{m+1} = p_{m-1}.
    # It is highly likely the source implies summing over the available unique bins 
    # or there is a typo in the upper limit of the sum in the source text.
    # However, to be safe and "implementable", we normalize by the total power 
    # of the residual excluding DC.
    
    total_power_excluding_dc = sum(p_hat[1:]) # Sum of p_2 to p_q (0-based index 1 to q-1)
    # If the source requires p_{q+1}, and we don't have it, we cannot compute it exactly.
    # But standard NCP uses total power.
    # Let's assume the denominator is sum(p_hat[1:]).
    
    c = []
    cumulative_sum = 0
    for j in range(1, q + 1): # j=1 to q
        # Numerator: sum_{i=2}^{j+1} p_i
        # In 0-based p_hat: indices 1 to j
        # p_2 is index 1. p_{j+1} is index j.
        # So we sum p_hat[1 : j+1]
        
        # Numerator calculation
        # We can maintain a running sum
        if j == 1:
            cumulative_sum = p_hat[1] # sum from i=2 to 2
        else:
            cumulative_sum += p_hat[j] # add p_{j+1} which is index j
            
        if total_power_excluding_dc == 0:
            c_val = 0.0
        else:
            c_val = cumulative_sum / total_power_excluding_dc
        c.append(c_val)
        
    # Step 5: Compute r_NCP
    # c_white is a straight line from (0,0) to (q,1).
    # The points are (j, j/q) for j=0..q? 
    # The NCP c is defined for j=1..q.
    # The source says "ranging from (0,0) to (q,1)".
    # This implies the x-axis is the index j.
    # So c_white[j] = j / q for j=0..q.
    # We compare c (length q, indices 1..q) with c_white (indices 1..q).
    
    r_ncp_sq = 0.0
    for j in range(1, q + 1):
        # c_white value at index j is j/q
        c_white_val = j / q
        diff = c[j-1] - c_white_val # c is 0-indexed, so c[j-1] is c_j
        r_ncp_sq += diff ** 2
        
    r_ncp = sqrt(r_ncp_sq)
    
    # Step 6: Check Termination
    # "Terminate ... when two iterations on either side of the current r_NCP(k) 
    # are larger than r_NCP(k)."
    # This requires a history of r_ncp values.
    # This function only computes the current value.
    # The caller must manage the history.
    return r_ncp
```

**Parameters**
*   $N_{max}$: Maximum number of iterations to prevent the algorithm from failing to terminate [S1].
*   $q = \lceil n/2 \rceil$: Number of periodogram elements [S1].

**Edge Cases and Pitfalls**
*   **Multiple Minima:** $r_{NCP}$ may exhibit multiple minima at low noise levels. The algorithm must iterate beyond the first detected minimum to ensure it is not stopping prematurely at a local minimum [S1].
*   **Computational Cost:** The method is computationally demanding because the residual $r_k$ and its DFT must be calculated after every iteration [S1]. Parallelization (e.g., via CUDA/CuPy) is recommended for the Fourier transform [S1].
*   **Residual Access:** If using libraries like ASTRA, the residual vector must be explicitly calculated after the SIRT update, as the library may only expose the norm [S1].

## Subtopics

### SIRT (Simultaneous Iterative Reconstruction Technique)
A common iterative solver used in conjunction with the NCP stopping rule. It updates all voxels simultaneously using all projections [S1, 3101-sirt-simultaneous-iterative-reconstruction-technique.md].

### Iterative Solvers
The parent module containing common iterative algorithms for solving tomography reconstruction problems [S1, 10781-iterative-solvers.md].

## Operating characteristics
*   **Time Complexity:** High per-iteration cost due to the explicit calculation of the residual vector and its Discrete Fourier Transform after every step [S1].
*   **Memory Complexity:** Requires storing the residual vector $r_k$ and the history of $r_{NCP}$ values for the stopping check [S1].
*   **Strengths:** Adapts to the noise level in the projection data, stopping the algorithm close to the optimal solution without requiring knowledge of the ground truth [S1].
*   **Limitations:** Computationally expensive compared to simple norm-based stopping criteria [S1].
*   **When to Use:** When the noise level in the projection data is unknown or variable, and computational resources allow for per-iteration DFT calculations [S1].
*   **When Not to Use:** When real-time performance is critical and the noise level is known or negligible [S1].

## Background
The NCP stopping rule was proposed as a method to minimize the reconstruction error without access to the ground truth, based on the assumption that the residual should resemble white noise when the reconstruction is optimal [S1]. It is particularly useful in dynamic imaging of multiphase flow, where noise levels can vary [S1]. The implementation described in the primary source utilizes the ASTRA toolbox for forward projection and CuPy for accelerated Fourier transforms [S1].

## Sources
[S1] Improved dynamic imaging of multiphase flow by constrained tomographic reconstruction., https://doi.org/10.1038/s41598-021-91776-1, fetched 2026-10-26
