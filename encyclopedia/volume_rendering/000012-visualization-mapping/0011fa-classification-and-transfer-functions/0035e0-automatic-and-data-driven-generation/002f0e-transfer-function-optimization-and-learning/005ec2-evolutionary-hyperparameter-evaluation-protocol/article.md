# Evolutionary Hyperparameter Evaluation Protocol

Also called: Elo-based Configuration Ranking, Population/Iteration Analysis, IntuiTF MLLM-Guided Optimization Process

## Overview
The Evolutionary Hyperparameter Evaluation Protocol is a systematic procedure to assess how evolutionary algorithm hyperparameters (population size, iteration count) affect transfer function optimization quality using pairwise Elo ratings [S1, unofficial]. It ranks transfer function candidates to determine optimal configuration settings and population parameters for MLLM-guided optimization [S1, unofficial]. Unlike methods that optimize the transfer function directly, this protocol evaluates the optimization process itself, identifying which hyperparameter configurations yield the highest-quality final transfer functions [S1, unofficial].

## How it works
**Inputs:**
*   A set of hyperparameter configurations $H = \{h_1, h_2, \dots, h_k\}$, where each $h_i$ defines a population size $m_i$ and an iteration limit $n_i$ [S1, unofficial].
*   An evolutionary optimizer capable of generating a population of transfer function candidates for a given configuration [S1, unofficial].
*   An MLLM-based evaluator (e.g., IntuiTF) capable of pairwise comparison of transfer functions [S1, unofficial].

**Outputs:**
*   A ranking of hyperparameter configurations based on the average Elo ranking of their best individuals [S1, unofficial].
*   Recommended population sizes and iteration counts based on computational budget constraints [S1, unofficial].

**Data Structures:**
*   **Configuration Pool:** A list of hyperparameter settings [S1, unofficial].
*   **Representative Pool:** A combined list of the top $K$ individuals (typically $K=10$) from the final population of each configuration [S1, unofficial].
*   **Elo Ratings:** A mapping from each individual ID to a scalar rating score [S1, unofficial].

**Steps:**
1.  **Run Optimizers:** For each hyperparameter configuration $h_i \in H$, run the evolutionary optimizer for $n_i$ iterations [S1, unofficial].
2.  **Extract Representatives:** From the final population of each configuration, select the top $K$ individuals (e.g., $K=10$) based on their fitness scores [S1, unofficial]. Let $R_i$ be the set of these $K$ individuals for configuration $h_i$ [S1, unofficial].
3.  **Combine Pool:** Create a collective pool $P = \bigcup_{i=1}^{k} R_i$ containing all representatives from all configurations [S1, unofficial].
4.  **Pairwise Comparison:** Apply the Elo rating method to the pool $P$ [S1, unofficial]. The source does not specify the exact pairing schedule, but states that each individual participates in $\log_2(m) + 2$ comparison rounds [S1, unofficial].
5.  **Calculate Average Ranking:** For each configuration $h_i$, calculate the average ranking of its representatives $R_i$ in the pool $P$ [S1, unofficial]. Let $Rank(x)$ be the rank of individual $x$ in the sorted list of all individuals in $P$ (1 being best) [S1, unofficial]. The score for configuration $h_i$ is:
    $$ Score(h_i) = \frac{1}{K} \sum_{x \in R_i} Rank(x) $$
6.  **Rank Configurations:** Rank the configurations $H$ by their scores $Score(h_i)$ [S1, unofficial]. Lower scores indicate superior optimization outcomes [S1, unofficial].

**Formulas:**
*   **Effective Cost per Iteration:** The computational cost for one iteration of the evaluation protocol is:
    $$ Cost_{iter} = (\log_2(m) + 2) \cdot T $$
    where $m$ is the population size and $T$ is the average API response time [S1, unofficial].
*   **Total Runtime:** The total runtime over $n$ iterations is:
    $$ TotalTime = n \cdot (\log_2(m) + 2) \cdot T \approx n \log_2(m) \cdot T $$
    The time complexity is $O(n \log_2 m)$ [S1, unofficial].

**Parameters:**
*   **Population Size ($m$):** Tested values: 10, 25, 50, 100, 200. Recommended: 25-50 for constrained resources; 100-200 for abundant resources [S1, unofficial].
*   **Iteration Count ($n$):** Recommended 20-30 for constrained resources. Quality stabilizes around 10-15 iterations [S1, unofficial].
*   **API Response Time ($T$):** Empirically measured at ~90 seconds per iteration for $m=50$ [S1, unofficial].
*   **Monetary Cost:** ~$0.23 USD per round of LLM API calls (using Gemini 2.0 Flash) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Small Populations:** Populations of size 10 and 25 improve rapidly in early stages but converge quickly with unstable rankings [S1, unofficial].
*   **Large Populations:** Require significantly more computational resources due to the $O(n \log_2 m)$ complexity of Elo rating calculations [S1, unofficial].
*   **Budget Constraints:** Under limited computational budgets, medium-small populations (25-50) perform better than larger ones [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evolutionary_hyperparameter_evaluation_protocol(configurations, k_representatives=10):
    """
    configurations: List of dicts, each with 'population_size' (m) and 'max_iterations' (n).
    k_representatives: Number of top individuals to extract from each final population.
    Returns: List of configurations sorted by average ranking (lowest first).
    """
    all_representatives = []
    config_to_reps = {}

    # Step 1 & 2: Run optimizers and extract representatives
    for config in configurations:
        m = config['population_size']
        n = config['max_iterations']
        
        # [G] run_evolutionary_optimizer is external to this protocol.
        # It initializes a population, performs crossover/mutation, and evaluates fitness.
        final_population = run_evolutionary_optimizer(m, n)
        
        # Sort by fitness and take top k
        final_population.sort(key=lambda ind: ind.fitness, reverse=True)
        representatives = final_population[:k_representatives]
        
        config_to_reps[config] = representatives
        all_representatives.extend(representatives)

    # Step 3 & 4: Combine pool and apply Elo rating
    # Initialize ratings
    ratings = {ind.id: 1500 for ind in all_representatives}
    
    # The source states each individual participates in log2(m) + 2 comparison rounds.
    # We use the maximum population size across configurations to determine rounds.
    max_m = max(c['population_size'] for c in configurations)
    num_rounds = int(math.log2(max_m) + 2)
    
    # Perform pairwise comparisons for the specified number of rounds.
    # Comparisons within a round execute in parallel.
    for _ in range(num_rounds):
        # Pair individuals sequentially for this round
        for i in range(0, len(all_representatives) - 1, 2):
            ind_a = all_representatives[i]
            ind_b = all_representatives[i + 1]
            
            # [G] get_mllm_preference is external to this protocol.
            # It queries the MLLM to compare two transfer functions.
            score_a = get_mllm_preference(ind_a, ind_b)
            score_b = 1.0 - score_a
            
            # Standard Elo update [G]
            expected_a = 1.0 / (1.0 + 10 ** ((ratings[ind_b.id] - ratings[ind_a.id]) / 400.0))
            ratings[ind_a.id] += 32 * (score_a - expected_a)
            ratings[ind_b.id] += 32 * ((1.0 - expected_a) - (1.0 - score_a))

    # Step 5: Calculate average ranking for each configuration
    # Rank individuals by rating (higher is better)
    sorted_individuals = sorted(all_representatives, key=lambda ind: ratings[ind.id], reverse=True)
    rank_map = {ind.id: rank + 1 for rank, ind in enumerate(sorted_individuals)}
    
    config_scores = []
    for config, reps in config_to_reps.items():
        avg_rank = sum(rank_map[ind.id] for ind in reps) / len(reps)
        config_scores.append((config, avg_rank))

    # Step 6: Rank configurations
    config_scores.sort(key=lambda x: x[1])
    return [config for config, score in config_scores]

def run_evolutionary_optimizer(population_size, max_iterations):
    """[G] External evolutionary optimizer implementation."""
    pass

def get_mllm_preference(ind_a, ind_b):
    """[G] External MLLM query implementation returning 1.0, 0.0, or 0.5."""
    pass
```

## Subtopics

### Elo-based Configuration Ranking
A method that uses pairwise comparisons and Elo ratings to rank hyperparameter configurations [S1, unofficial]. It sets itself apart by providing a relative quality metric rather than absolute fitness scores, which is useful when absolute metrics are noisy or unavailable [S1, unofficial]. Choose this when comparing multiple optimization strategies where relative performance is key [S1, unofficial]. [Link](52-elo-based-configuration-ranking.md)

### Population/Iteration Analysis
A study of how population size and iteration count affect optimization outcomes [S1, unofficial]. It sets itself apart by focusing on the resource-quality trade-off [S1, unofficial]. Choose this when computational budget is a primary constraint [S1, unofficial]. [Link](53-population-iteration-analysis.md)

## Operating characteristics
*   **Time Complexity:** $O(n \log_2 m)$, where $n$ is the iteration count and $m$ is the population size [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** For a population of 50, each iteration takes approximately 90 seconds, primarily due to external API communication [S1, unofficial].
*   **Scaling:** Larger populations (100-200) achieve more stable improvements with continued iterations, while smaller populations (10-25) show early rapid gains but unstable convergence [S1, unofficial].
*   **Strengths:** Provides a robust way to compare optimization configurations using relative rankings [S1, unofficial]. Effective under limited computational resources when using medium-small populations (25-50) [S1, unofficial].
*   **Limitations:** High computational cost due to MLLM API calls [S1, unofficial]. Small populations may converge prematurely with unstable rankings [S1, unofficial].
*   **When to Use:** When optimizing transfer functions using MLLMs and computational resources are limited [S1, unofficial]. Best for selecting hyperparameters before full-scale optimization [S1, unofficial].
*   **When Not to Use:** When real-time optimization is required, or when computational budget is extremely high and larger populations are feasible without cost concerns [S1, unofficial].

## Background
The protocol was introduced in the IntuiTF framework for MLLM-guided transfer function optimization in direct volume rendering [S1, unofficial]. It addresses the challenge of selecting optimal hyperparameters for evolutionary algorithms when the fitness evaluation is expensive and relies on large language models [S1, unofficial]. The method leverages the Elo rating system, commonly used in chess and gaming, to rank transfer function candidates [S1, unofficial].

## Sources
[S1] IntuiTF: MLLM-Guided Transfer Function Optimization for Direct Volume Rendering, https://arxiv.org/abs/2506.18407v2, fetched 2026-10-26 (unofficial)
