# PCP from 2D Histograms

Also called: Novotny and Hauser PCP Generation

## Overview
This method separates the number of polylines rendered in a Parallel Coordinate Plot (PCP) from the number of raw data samples by generating the plot from 2D histograms of adjacent variable pairs [S1, unofficial]. It addresses the over-plotting and prohibitively expensive rendering costs associated with large datasets [S1, unofficial]. Unlike standard PCPs where each sample is a polyline, this approach aggregates data into histograms to define the visual representation, while also enabling outlier identification through histogram analysis [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset with $N$ samples and $D$ attributes (variables). Adjacent attribute pairs $(x_k, x_{k+1})$ for $k=1 \dots D-1$.
*   **Output:** A set of polylines representing the PCP.

**Data Structures:**
*   **2D Histogram:** For each adjacent pair of attributes $(x, y)$, a 2D histogram $H$ is computed. The histogram bins represent the joint frequency distribution of the two attributes.
    *   Let $B_x$ be the number of bins for attribute $x$.
    *   Let $B_y$ be the number of bins for attribute $y$.
    *   The histogram is a matrix $H \in \mathbb{Z}^{B_x \times B_y}$, where $H(i, j)$ stores the count of samples falling into bin $i$ of $x$ and bin $j$ of $y$.

**Steps:**
1.  **Histogram Generation:** For each adjacent pair of attributes in the dataset, compute the 2D joint histogram. Normalize or bin the attribute values into discrete intervals to form the histogram grid [S1, unofficial].
2.  **Quad Mapping:** Identify all non-zero pixels $P(i, j)$ in the joint histogram of an attribute pair $(x, y)$ [Parent]. Each non-zero pixel corresponds to a bin intersection containing data samples.
3.  **Polylines Construction:** For each non-zero pixel $P(i, j)$ in the histogram of the pair $(x_k, x_{k+1})$, generate a line segment (or quad as per parent definition) connecting position $i$ on the PC axis for $x_k$ to position $j$ on the PC axis for $x_{k+1}$ [Parent].
    *   The position $i$ on axis $x_k$ corresponds to the center or representative value of bin $i$.
    *   The position $j$ on axis $x_{k+1}$ corresponds to the center or representative value of bin $j$.
4.  **Aggregation:** Repeat for all adjacent attribute pairs. The resulting set of segments forms the PCP. The number of polylines is determined by the number of non-zero histogram bins, not the number of data samples $N$ [S1, unofficial].
5.  **Outlier Identification:** Analyze the histogram values $H(i, j)$. Samples or bin combinations with extremely low counts relative to the distribution may be identified as outliers [S1, unofficial].

**Parameters:**
*   **Bin Count:** The number of bins $B_x, B_y$ for each attribute. Not given in the sources.
*   **Normalization:** Whether histogram counts are normalized. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparsity:** If the histogram is very sparse, many bins may be zero, reducing the number of polylines significantly.
*   **Bin Resolution:** Too few bins may obscure important correlations; too many bins may reintroduce over-plotting or noise [S1, unofficial].
*   **Outlier Detection:** The definition of an outlier based on histogram analysis is not explicitly defined in the sources [S1, unofficial].

## Subtopics

### Visual Clustering on PCPs
A method to perform visual clustering on the PCPs by drawing curved edges instead of polylines and optimizing the arrangement of these curved edges [S1, unofficial]. Zhou et al. [102] propose this approach to enhance clustering visualization [S1, unofficial].

### Parallel Coordinate Plot Clustering
A method to address over-plotting in PCPs by clustering the data values and rendering only the representing polylines of each cluster [S1, unofficial]. Fua et al. [26] propose clustering data values and rendering only representative polylines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the histogram computation $O(N \cdot D)$ and the number of non-zero histogram bins. The rendering cost is proportional to the number of non-zero bins, which is typically much smaller than $N$ [S1, unofficial].
*   **Memory Complexity:** Proportional to the size of the 2D histograms, $O(D \cdot B_x \cdot B_y)$ [S1, unofficial].
*   **Strengths:** Separates rendering cost from data size; reduces over-plotting; enables outlier identification [S1, unofficial].
*   **Limitations:** Loses individual sample information; relies on binning which may introduce artifacts [S1, unofficial].
*   **When to use:** Large datasets where over-plotting is a concern and global trends are more important than individual samples [S1, unofficial].
*   **When not to use:** When individual sample identification is required [S1, unofficial].

## Background
Novotny and Hauser [70] introduced the method of generating PCPs from 2D histograms to separate the number of polylines from the number of data samples [S1, unofficial]. This approach is part of the broader effort to address over-plotting in multivariate visualization [S1, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
