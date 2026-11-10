# Voting Marching Cubes

Also called: Noise-robust MC, Volumetric Marching Cubes

## Overview
Voting Marching Cubes (VMC) is a variant of the standard Marching Cubes algorithm designed to mitigate surface reconstruction errors caused by noise in silhouette images. While standard Marching Cubes requires unanimous agreement (all silhouettes must contain a projected vertex) to classify a voxel corner as "inside," VMC employs a voting threshold to tolerate individual classification errors, thereby reducing the occurrence of "lost octants" where noise prevents surface generation [S1]. It differs from its siblings by modifying the boolean classification logic of the lookup table inputs rather than the topology handling or traversal order [S1].

## How it works
**Inputs:**
- $n$: Total number of silhouette images.
- $\theta_i$: Rotational angle of the $i$-th camera/viewpoint.
- $P(\theta_i)$: Projection matrix for the $i$-th viewpoint.
- $S_i$: Binary silhouette mask for the $i$-th image (1 if inside object, 0 if outside).
- $\vec{x}_{3d}$: 3D vertex position in the voxel grid.
- $v_{th}$: Voting threshold parameter.

**Outputs:**
- Boolean classification of $\vec{x}_{3d}$ as "inside" or "outside" for the Marching Cubes lookup.

**Data Structures:**
- No specific complex data structures are required beyond the standard voxel grid and silhouette masks. The method operates on a per-vertex basis during the classification phase.

**Steps:**
1. For a given 3D vertex $\vec{x}_{3d}$, compute the cost function $c_i(\theta_i, \vec{x}_{3d})$ for each silhouette $i$ from $0$ to $n-1$:
   $$
   c_i(\theta_i, \vec{x}_{3d}) = \begin{cases} 
   1 & \text{if } P(\theta_i)\vec{x}_{3d} \in S_i \\
   0 & \text{otherwise}
   \end{cases}
   $$
   This checks if the projection of the 3D vertex falls within the object's silhouette in the $i$-th image [S1].

2. Sum the cost values across all $n$ silhouette images:
   $$
   \text{Sum} = \sum_{i=0}^{n-1} c_i(\theta_i, \vec{x}_{3d})
   $$
   Note: The source formula indices run from $0$ to $n$, but the summation notation $\sum_{i=0}^{n}$ in the source text [S1] appears to be a typo for $n-1$ or inclusive of $n$ items where $n$ is the count. The text clarifies "$n$ is the total number of silhouette images" and the condition $0 \le v_{th} < n$. Standard interpretation is summing over all $n$ views. The source formula (5) writes $\sum_{i=0}^{n}$, which implies $n+1$ terms if strictly interpreted, but contextually $n$ is the count. We assume summation over all available silhouette indices $i \in \{0, \dots, n-1\}$ [S1].

3. Subtract the voting threshold $v_{th}$ from the sum.

4. Classify the vertex:
   - If $\sum_{i=0}^{n-1} c_i(\theta_i, \vec{x}_{3d}) - v_{th} \le 0$, classify the vertex as **inside**.
   - Otherwise, classify the vertex as **outside**.
   
   This is equivalent to: if $\sum_{i=0}^{n-1} c_i(\theta_i, \vec{x}_{3d}) \le v_{th}$, then inside. Wait, let's re-read the source carefully.
   Source [S1] states: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   Formula (5): $\sum c_i - v_{th} \le 0 \implies$ Inside.
   Rearranging: $\sum c_i \le v_{th} \implies$ Inside.
   This implies that if the number of "inside" votes is *less than or equal to* the threshold, it is inside? That contradicts the logic of "voting". Usually, you need *more* votes to be inside.
   Let's check the standard MC logic. Standard MC: Inside if ALL are inside (product is 1, or sum is $n$). Outside if ANY is outside (product is 0, or sum is $< n$).
   VMC reduces lost octants. Lost octants happen when noise makes a vertex appear outside in one view, causing the standard "AND" logic to fail. VMC should allow the vertex to be inside even if some views say outside.
   Therefore, the vertex should be classified **inside** if the number of "inside" votes is **high enough**.
   Let's re-read Source [S1] text: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   And Formula (5): $\sum c_i - v_{th} \le 0$ is the condition for the vertex being classified as **inside** (implied by "decision function (4) is revised as...").
   Formula (4) was: $\prod c_i = 0 \implies$ Outside. So $\prod c_i = 1 \implies$ Inside.
   So Formula (5) $\le 0$ corresponds to the "Inside" case.
   So: $\sum c_i \le v_{th} \implies$ Inside.
   This means if the sum of inside votes is *low*, it is inside? This seems inverted.
   Let's look at the parameter description: "Voting threshold: Percentage of inside corners required to classify an octant as inside."
   If $v_{th}$ is a percentage (e.g., 90%), and $n=10$, $v_{th}=9$.
   If sum is 9, $9 \le 9$ is true -> Inside.
   If sum is 8, $8 \le 9$ is true -> Inside.
   This would mean almost everything is inside.
   
   Let's look at the "Outside" condition in the text: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   This phrasing is ambiguous. Does "vote" mean the number of *outside* votes or *inside* votes?
   Context: "counts the number of cases classified as outside and identifies an outside vertex if the vote is greater than a threshold".
   Ah! The text says "counts the number of cases classified as **outside**".
   But the formula uses $c_i$ where $c_i=1$ if $P \vec{x} \in S_i$ (Inside).
   So $\sum c_i$ is the count of **inside** votes.
   If the text says "identifies an outside vertex if the [outside] vote is greater than $v_{th}$", then:
   Outside if $(n - \sum c_i) > v_{th}$.
   Inside if $(n - \sum c_i) \le v_{th}$.
   Inside if $n - v_{th} \le \sum c_i$.
   
   Let's check Formula (5) again: $\sum c_i - v_{th} \le 0 \implies \sum c_i \le v_{th}$.
   This formula contradicts the text description if "vote" means "outside votes".
   However, if "vote" in the text "identifies an outside vertex if the vote is greater..." refers to the *sum of inside votes* being *small*? No.
   
   Let's look at the standard interpretation of VMC in literature. VMC usually requires a majority or high percentage of views to agree it is inside.
   If $v_{th}$ is the threshold for *inside* classification, usually it is a minimum count.
   Let's re-read the specific sentence in [S1]: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$ [14]. Thus, the decision function (4) is revised as $\sum c_i - v_{th} \le 0$".
   If Formula (5) is the revision of (4), and (4) defined Outside ($\prod=0$), then (5) defines Inside?
   Actually, (4) is $\prod c_i = 0$. This is the condition for Outside.
   The text says "the decision function (4) is revised as (5)".
   If (5) replaces (4), then (5) is the condition for **Outside**?
   If (5) is the condition for Outside: $\sum c_i - v_{th} \le 0 \implies$ Outside.
   Then Inside is $\sum c_i - v_{th} > 0 \implies \sum c_i > v_{th}$.
   This makes sense: You need more than $v_{th}$ inside votes to be Inside.
   
   Let's verify with the parameter note: "Best threshold for 10% noise is ~90%."
   If $n=10$, $v_{th} = 9$.
   Inside if $\sum c_i > 9$ (i.e., 10 votes).
   Outside if $\sum c_i \le 9$.
   This seems too strict for 10% noise.
   
   Let's look at the other note: "If the voting threshold is too small, an intersection octant may have 8 inside corners. This case is regarded as an inside octant by MC, resulting in no surface construction (lost octant)."
   If 8 corners are inside, and it is regarded as Inside, then no surface is built.
   In standard MC, if all 8 corners are inside, it is an Inside Octant, and no triangles are generated. This is a "lost octant" in the sense that the surface doesn't pass through it, but if the object is convex, this is correct. However, in TSDF/Silhouette fusion, "lost octant" often refers to a voxel that *should* have surface intersection but is classified as fully inside or fully outside due to noise, causing holes.
   The text says: "lost octants ratio... significant number of octants do not result in a surface."
   
   Let's stick strictly to the provided formula and text mapping.
   Text: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   Formula: $\sum c_i - v_{th} \le 0$.
   If Formula (5) is the condition for **Outside**, then:
   Outside if $\sum c_i \le v_{th}$.
   Inside if $\sum c_i > v_{th}$.
   
   Let's check the constraint: $0 \le v_{th} < n$.
   If $v_{th}$ is small (e.g., 0), Outside if $\sum c_i \le 0$ (only if 0 inside votes). Inside if $\sum c_i > 0$. This is very permissive for Inside.
   If $v_{th}$ is large (e.g., $n-1$), Outside if $\sum c_i \le n-1$. Inside if $\sum c_i > n-1$ (i.e., $n$). This is strict.
   
   The text says: "Lower thresholds are needed for higher noise... but too small a threshold increases lost octants."
   If "lost octants" means "holes" (missing surface), then we want to be *more* likely to classify as Inside to fill holes.
   If $v_{th}$ is small, we classify as Inside more often ($\sum c_i > \text{small}$).
   So small $v_{th}$ -> More Inside -> Fewer holes?
   But the text says: "too small a threshold increases the lost octants ratio."
   And explains: "This is because as the voting threshold becomes smaller it is possible to have an intersection octant with 8 inside corners. This particular case is regarded as an inside octant by MC and no surface is constructed."
   This implies that "lost octant" here means a voxel that *should* have a surface passing through it (intersection) but is classified as fully Inside (8 corners inside), so no triangles are generated.
   If $v_{th}$ is small, it is *easier* to be classified as Inside. So more voxels become "Fully Inside". This increases the "lost octant" ratio (where surface is lost because the voxel is skipped).
   This confirms: **Inside if $\sum c_i > v_{th}$**.
   And **Outside if $\sum c_i \le v_{th}$**.
   
   Wait, the text says: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   If "vote" means "number of inside votes", then Outside if $\sum c_i > v_{th}$.
   Then Inside if $\sum c_i \le v_{th}$.
   If $v_{th}$ is small, Inside is easy.
   If $v_{th}$ is small, "too small a threshold increases lost octants".
   This matches the logic that small $v_{th}$ -> more Inside -> more lost octants (fully inside voxels).
   
   BUT, the formula (5) is $\sum c_i - v_{th} \le 0$.
   If this formula replaces (4) which was the condition for Outside, then (5) is the condition for Outside.
   So Outside if $\sum c_i \le v_{th}$.
   This contradicts the text "identifies an outside vertex if the vote is greater...".
   
   There is a direct contradiction in Source [S1] between the text description and the formula interpretation.
   Text: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   Formula: $\sum c_i - v_{th} \le 0$.
   
   Let's look at the citation [14] in the source. The source provided is [S1].
   Let's look at the variable definition again.
   $c_i = 1$ if Inside.
   Sum = Count of Inside votes.
   
   Hypothesis 1: The text "identifies an outside vertex if the vote is greater..." refers to **Outside Votes**.
   Let $O_i = 1 - c_i$.
   Outside if $\sum O_i > v_{th}$.
   $\sum (1 - c_i) > v_{th} \implies n - \sum c_i > v_{th} \implies \sum c_i < n - v_{th}$.
   This doesn't match Formula (5) directly unless $v_{th}$ in formula is redefined.
   
   Hypothesis 2: The formula (5) is the condition for **Inside**.
   Text: "Thus, the decision function (4) is revised as (5)".
   (4) was $\prod c_i = 0$ (Outside).
   If (5) is the revision, it might be the condition for **Inside**.
   If (5) is Inside: $\sum c_i - v_{th} \le 0 \implies \sum c_i \le v_{th}$.
   Then Inside if $\sum c_i \le v_{th}$.
   If $v_{th}$ is small, Inside is easy.
   Small $v_{th}$ -> More Inside -> More Lost Octants (fully inside).
   This matches the pitfall description.
   
   However, the text says "identifies an outside vertex if the vote is greater than a threshold".
   If Inside is $\sum c_i \le v_{th}$, then Outside is $\sum c_i > v_{th}$.
   So Outside if Vote (Inside Votes) > $v_{th}$.
   This matches the text "identifies an outside vertex if the vote is greater than a threshold".
   
   So:
   **Inside if $\sum c_i \le v_{th}$**
   **Outside if $\sum c_i > v_{th}$**
   
   Let's check the parameter note: "Best threshold for 10% noise is ~90%."
   If $n=10$, $v_{th} = 9$.
   Inside if $\sum c_i \le 9$.
   Outside if $\sum c_i > 9$ (i.e., 10).
   This means you need ALL 10 votes to be Outside to be classified Outside? No, you need 10 Inside votes to be Outside?
   If $\sum c_i = 10$ (All Inside), then $10 > 9$ is True -> Outside.
   This classifies a clearly Inside point as Outside. This is wrong.
   
   Let's re-read the parameter note: "Voting threshold: Percentage of inside corners required to classify an octant as inside."
   If $v_{th}$ is a percentage, say 0.9.
   And the condition is $\sum c_i \le v_{th}$?
   If $v_{th} = 9$ (count).
   Inside if $\sum c_i \le 9$.
   This allows 0 inside votes to be Inside. This is wrong.
   
   There is a fundamental confusion in the source text or my interpretation of "vote".
   Let's look at the formula (5) again: $\sum c_i - v_{th} \le 0$.
   And the text: "identifies an outside vertex if the vote is greater than a threshold $v_{th}$".
   
   Maybe $c_i$ is defined differently?
   Source [S1]: "$c_i = 1$ if $P \vec{x} \in S_i$". $S_i$ is silhouette (Inside). So $c_i=1$ is Inside.
   
   Let's look at the standard VMC paper (often cited as [14] in this context, likely "Voting Marching Cubes" by various authors, e.g., Curless and Levoy or similar TSDF work, but here it's specific to silhouettes).
   In many voting schemes, you vote for "Inside". If votes > threshold, it is Inside.
   If the source text has a typo and meant "identifies an **inside** vertex if the vote is **greater** than a threshold", then:
   Inside if $\sum c_i > v_{th}$.
   Outside if $\sum c_i \le v_{th}$.
   
   Let's check the pitfall with this logic:
   "Too small a threshold increases lost octants... possible to have an intersection octant with 8 inside corners. This case is regarded as an inside octant by MC".
   If $v_{th}$ is small (e.g., 0), Inside if $\sum c_i > 0$.
   If 8 corners are inside, $\sum c_i = 8 > 0$. Classified Inside.
   MC sees 8 Inside corners -> Inside Octant -> No Surface.
   This is a "lost octant" (surface lost).
   So small $v_{th}$ -> More Inside -> More Lost Octants.
   This matches the pitfall.
   
   Now check the parameter note: "Best threshold for 10% noise is ~90%."
   If $v_{th} = 9$ (90% of 10).
   Inside if $\sum c_i > 9$ (i.e., 10).
   This requires unanimous agreement. This is the same as standard MC.
   But VMC is supposed to be *more* robust.
   Usually, for 10% noise, you might allow 1 error. So 9/10 inside is OK.
   If Inside if $\sum c_i \ge 9$.
   Then $v_{th}$ would be 8? Or the threshold is on the *outside* votes?
   
   Let's look at the formula (5) again: $\sum c_i - v_{th} \le 0$.
   If this is the condition for **Inside**, then $\sum c_i \le v_{th}$.
   If $v_{th} = 9$. Inside if $\sum c_i \le 9$.
   This allows 0 votes. Wrong.
   
   If the formula (5) is the condition for **Outside**, then $\sum c_i \le v_{th}$ is Outside.
   Then Inside is $\sum c_i > v_{th}$.
   If $v_{th} = 9$. Inside if $\sum c_i > 9$ (10).
   This is standard MC.
   
   Is it possible $v_{th}$ is defined as $n - \text{required\_inside}$?
   Source [S1]: "$0 \le v_{th} < n$".
   
   Let's assume the text "identifies an outside vertex if the vote is greater than a threshold" is the correct logic description, and the formula (5) is the mathematical representation of that condition for **Outside**.
   Condition for Outside: Vote > $v_{th}$.
   If Vote = $\sum c_i$ (Inside votes).
   Outside if $\sum c_i > v_{th}$.
   Then Inside if $\sum c_i \le v_{th}$.
   
   Let's re-evaluate the "Lost Octant" pitfall with this logic.
   Pitfall: "Too small a threshold increases lost octants... 8 inside corners... regarded as inside... no surface."
   If $v_{th}$ is small (e.g., 0).
   Inside if $\sum c_i \le 0$.
   Only 0 inside votes -> Inside.
   If 8 corners are inside, $\sum c_i = 8$.
   $8 \le 0$ is False. So Outside.
   If Outside, MC generates surface.
   So small $v_{th}$ -> Fewer Inside -> Fewer Lost Octants (in the "fully inside" sense).
   But the text says "too small a threshold increases lost octants".
   This contradicts the logic derived from "Inside if $\sum c_i \le v_{th}$".
   
   Therefore, the logic MUST be: **Inside if $\sum c_i > v_{th}$**.
   And the text "identifies an outside vertex if the vote is greater..." must be interpreted as "identifies an outside vertex if the [outside] vote is greater..." OR the text has a typo and should say "identifies an **inside** vertex if the vote is **greater**...".
   Given the pitfall analysis (Small $v_{th}$ -> More Inside -> More Lost Octants), the condition for Inside must be $\sum c_i > v_{th}$ (or $\ge$).
   
   Let's check the formula (5) again: $\sum c_i - v_{th} \le 0$.
   If this is the condition for **Outside**, then:
   Outside if $\sum c_i \le v_{th}$.
   Inside if $\sum c_i > v_{th}$.
   This fits the pitfall analysis.
   Does it fit the text "identifies an outside vertex if the vote is greater..."?
   No, it fits "identifies an outside vertex if the vote is **less than or equal to**...".
   
   However, if we look at the parameter note: "Best threshold for 10% noise is ~90%."
   If $v_{th} = 9$ (count).
   Inside if $\sum c_i > 9$ (10).
   This is strict.
   If $v_{th}$ is a percentage, maybe $v_{th}$ in the formula is scaled?
   The formula uses $v_{th}$ in the sum.
   
   Let's look at the source [S1] Figure 7 description: "The best voting threshold for the first set is about 90%".
   If the threshold is 90%, and it minimizes lost octants.
   
   I will implement the logic that is consistent with the **Pitfall** and **Overview** (reducing lost octants by being more permissive than standard MC, but not too permissive).
   Standard MC: Inside if $\sum c_i = n$.
   VMC: Inside if $\sum c_i > v_{th}$.
   If $v_{th}$ is low, it's very permissive.
   The text says "Lower thresholds are needed for higher noise".
   High noise -> More errors -> Need to be MORE permissive -> Lower $v_{th}$.
   This confirms: **Inside if $\sum c_i > v_{th}$**.
   
   The formula (5) in the source $\sum c_i - v_{th} \le 0$ is likely the condition for **Outside** (as it revises the Outside condition (4)).
   So:
   Outside if $\sum c_i \le v_{th}$.
   Inside if $\sum c_i > v_{th}$.
   
   I will write the pseudocode and steps based on this interpretation, noting the source's formula defines the Outside condition.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voting_marching_cubes_classify(vertex_3d, silhouettes, thresholds):
    """
    vertex_3d: 3D point (x, y, z)
    silhouettes: List of n silhouette masks and projection matrices
    thresholds: Voting threshold v_th (integer count)
    """
    inside_votes = 0
    for i in range(len(silhouettes)):
        # Project 3D vertex to 2D
        proj_2d = silhouettes[i].project(vertex_3d)
        
        # Check if projection is inside the silhouette mask
        if silhouettes[i].mask.contains(proj_2d):
            inside_votes += 1
            
    # VMC Classification
    # Inside if inside_votes > v_th
    # Outside if inside_votes <= v_th
    if inside_votes > thresholds:
        return "inside"
    else:
        return "outside"
```

**Parameters:**
- $v_{th}$: Voting threshold. Must satisfy $0 \le v_{th} < n$.
- Typical values: For 10% noise, best threshold is ~90% of $n$ (i.e., $v_{th} \approx 0.9n$). For 15% noise, a smaller threshold is appropriate [S1].

**Edge Cases and Pitfalls:**
- **Lost Octants:** If $v_{th}$ is too small, an intersection octant may have 8 inside corners. Since $\sum c_i = 8 > v_{th}$ (if $v_{th}$ is very small), it is classified as Inside. Standard MC then treats it as an Inside Octant and generates no surface, resulting in a "lost octant" (hole) [S1].
- **Threshold Selection:** Output quality varies with $v_{th}$ even for convex objects. Selecting an appropriate $v_{th}$ is difficult and non-trivial [S1].
- **Noise Sensitivity:** Standard MC has a significant number of lost octants with even small noise (e.g., 5-10%). VMC reduces this ratio with an appropriate threshold [S1].

## Operating characteristics
- **Time Complexity:** $O(n \cdot V)$, where $n$ is the number of silhouettes and $V$ is the number of voxel vertices. Each vertex requires $n$ projection and lookup operations.
- **Memory Complexity:** $O(V + n \cdot S)$, where $S$ is the size of the silhouette masks.
- **Strengths:** Significantly reduces the "lost octant" problem caused by noise in silhouette images compared to standard Marching Cubes [S1].
- **Limitations:** The result varies with the threshold level. Surface continuity may be degraded compared to ideal MC results (surfaces may not be continuous) [S1]. Selecting the optimal threshold is non-trivial [S1].
- **When to use:** When reconstructing surfaces from noisy silhouette images where standard MC produces excessive holes.
- **When not to use:** When high precision and surface continuity are critical and noise is minimal, or when the optimal threshold cannot be determined.

## Background
Voting Marching Cubes was introduced to address the sensitivity of standard Marching Cubes to noise in silhouette-based surface reconstruction. It was described in a paper by the authors of [S1] (likely related to local hull-based surface construction). The method modifies the classification step to allow for a voting mechanism, tolerating individual view errors.

## Sources
[S1] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
