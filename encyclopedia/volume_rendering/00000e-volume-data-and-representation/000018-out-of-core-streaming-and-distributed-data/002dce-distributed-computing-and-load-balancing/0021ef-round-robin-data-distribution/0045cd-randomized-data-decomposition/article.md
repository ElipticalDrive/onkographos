# Randomized Data Decomposition

Also called: Probabilistic load balancing, Random block assignment

## Overview
Randomized Data Decomposition is a data distribution method where blocks are randomly assigned to processors with equal probability, relying on statistical theorems to guarantee load balance with high probability [S1]. It is primarily used in parallel and out-of-core isocontouring of massive datasets where static load estimation is difficult or expensive [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** $N$ data blocks ($b_1, \dots, b_N$) with $N \gg P$; $P$ processors ($p_1, \dots, p_P$); workload $w_i$ for each block $b_i$ at a given isovalue $\nu$; maximum block workload $\hat{w} = \max_{i=1}^N w_i$; total workload $W = \sum_{i=1}^N w_i$.
- **Outputs:** Assignment map specifying the destination processor for each block; distributed data blocks on their assigned processors.
- **Data Structures:** Not given in the sources.

**Algorithm Steps:**
1. Each processor determines the destination processor for each of its local blocks by sampling uniformly from $\{1, \dots, P\}$ with probability $1/P$ [S1].
2. Each processor collects blocks destined for other processors and transmits them to their destinations in rounds [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def randomized_data_decomposition(blocks, workloads, num_processors):
    """
    Assigns blocks to processors using randomized decomposition.
    Args:
        blocks: List of block identifiers [b1, ..., bN]
        workloads: List of workload values [w1, ..., wN]
        num_processors: Integer P
    Returns:
        assignments: List of destination processor indices [j1, ..., jN]
    """
    n = len(blocks)
    assignments = []
    for i in range(n):
        # Step 1: Determine destination uniformly at random
        dest = random.randint(0, num_processors - 1)
        assignments.append(dest)
    return assignments

def distribute_blocks(blocks, assignments, num_processors):
    """
    Collects and routes blocks to destination processors.
    Args:
        blocks: List of block identifiers
        assignments: List of destination processor indices
        num_processors: Integer P
    Returns:
        processor_lists: List of lists, where processor_lists[k] contains
                         blocks assigned to processor k
    """
    # Step 2: Collect blocks for each processor
    processor_lists = [[] for _ in range(num_processors)]
    for i in range(len(blocks)):
        dest = assignments[i]
        processor_lists[dest].append(blocks[i])
    return processor_lists
```

**Formulas and Probabilistic Guarantees:**
Let $x_{ij}$ be a binary random variable such that $x_{ij} = 1$ if block $b_i$ is assigned to processor $p_j$, and $0$ otherwise [S1]. The variables $x_{1j}, x_{2j}, \dots, x_{Nj}$ form a sequence of independent Bernoulli trials with $P(x_{ij}=1) = 1/P$ [S1]. The expected number of blocks received by processor $p_j$ is:
$$ E\left(\sum_{i=1}^N x_{ij}\right) = \frac{N}{P} $$
Let $a_i = w_i / \hat{w}$ be the normalized workload of block $b_i$, where $a_i \in (0, 1]$ [S1]. The normalized workload assigned to processor $p_j$ is $\Psi_j = \sum_{i=1}^N a_i x_{ij}$ [S1]. The expected workload assigned to processor $p_j$ is:
$$ \hat{w} \times E(\Psi_j) = \sum_{i=1}^N \frac{w_i}{P} = \frac{W}{P} $$
**Theorem 1 (Raghavan and Spencer [45]):** Let $a_1, \dots, a_N$ be reals in $(0, 1]$. Let $x_1, \dots, x_N$ be independent Bernoulli trials with $E(x_i) = \rho_i$. Let $\Psi_\beta = \sum_{i=1}^N a_i x_i$. If $E(\Psi_\beta) > 0$, then for any $\nu > 0$:
$$ \text{prob}(\Psi_\beta > (1+\nu)E(\Psi_\beta)) < \left( \frac{e^\nu}{(1+\nu)^{(1+\nu)}} \right)^{E(\Psi_\beta)} $$
**Theorem 2:** If the total workload $W > \alpha \hat{w} P \log r$ for a given isovalue and blocks are randomly allocated among $P$ processors, any processor has a workload $\le \frac{2W}{P}$ with high probability $\ge (1 - \frac{1}{r^c})$, where $\hat{w}$ is the maximum workload of a block and $\alpha > \frac{c+1}{\log(4/e)}$ ($c \le 0$) is a constant [S1]. The probability of any processor having more than twice the average workload is:
$$ \text{prob}(\Psi > 2\frac{W'}{P}) \le P \times \text{prob}(\Psi_j > 2\frac{W'}{P}) < r^{-c} $$
where $\Psi = \max_{j=1}^P (\Psi_j)$ is the maximum load across all processors, $W' = \sum_{i=1}^N a_i$ is the normalized total workload, and $r \ge P$ [S1].

**Edge Cases and Pitfalls:**
- **Small Workload:** The probabilistic guarantees assume $N \gg P$ and hold except for isosurfaces with very few triangles (low total workload), which are typically of little interest [S1].
- **Precision:** Normalization $a_i = w_i / \hat{w}$ requires floating-point arithmetic; extreme dynamic range in $w_i$ may affect bound tightness, though the assignment logic remains integer-based [G].
- **Randomness:** Reproducibility requires deterministic seeding of the random number generator [G].

## Operating characteristics
- **Load Balance:** Achieves good load balance with high probability for large datasets where $N \gg P$ [S1].
- **Time and Memory Complexity:** Not given in the sources.
- **Measured Performance and Scaling:** Not given in the sources.
- **Strengths and Limitations:** Simple to implement and requires no static load estimation [S1]. Does not preserve spatial locality, which may impact cache or I/O efficiency unless mitigated by subsequent indexing structures [S1].
- **When to Use/Not Use:** Suitable for large-scale parallel and out-of-core isocontouring where static balancing is expensive or inaccurate [S1]. Not suitable for small datasets where statistical averages do not hold [S1].

## Background
Randomized Data Decomposition was proposed for scalable isosurface visualization of massive datasets on commodity off-the-shelf clusters [S1]. It relies on statistical theorems regarding the weighted sum of Bernoulli trials, specifically citing work by Raghavan and Spencer [45] [S1]. The method addresses load balancing in parallel and out-of-core isocontouring, where block workloads vary significantly with the isovalue [S1]. It assumes $N \gg P$, a condition typical for massive volume datasets [S1].

## Sources
[S1] Scalable isosurface visualization of massive datasets on commodity off-the-shelf clusters., https://doi.org/10.1016/j.jpdc.2008.07.006, fetched 2026-11-08
