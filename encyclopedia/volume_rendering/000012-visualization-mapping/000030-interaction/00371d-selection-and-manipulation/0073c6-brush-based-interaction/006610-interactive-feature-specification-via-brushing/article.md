# Interactive Feature Specification via Brushing

Also called: Brushing in n-dimensional domain, Composite non-binary brushes

## Overview
Interactive Feature Specification via Brushing is a technique for defining complex data features by selecting subsets of data points across multiple linked visualizations, such as scatterplots and histograms [S1, unofficial]. It solves the problem of intuitively identifying high-dimensional structures in simulation data by allowing users to iteratively refine selections in one view while observing immediate effects in all others [S1, unofficial]. Unlike binary selection methods, it uses composite non-binary brushes to assign a continuous degree of interest (DOI) to every sample point, enabling the identification of complex, overlapping features [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** High-dimensional simulation data (e.g., computational flow simulation data) with attributes such as absolute pressure, velocity, viscosity, and turbulence kinetic energy [S1, unofficial].
*   **Output:** A Feature Definition Language (FDL) structure representing the selected features, and a smooth Degree of Interest (DOI) function mapping every sample point to a floating-point value between 0 and 1 [S1, unofficial].

**Data Structures:**
The core data structure is the Feature Definition Language (FDL), a hierarchical tree [S1, unofficial]:
1.  **Root Layer:** Contains the filename and an arbitrary number of `FeatureSet` objects [S1, unofficial].
2.  **Feature Set:** A group of `Feature` objects that are to be shown simultaneously. Users can switch between, add, delete, or move features between sets [S1, unofficial].
3.  **Feature Layer:** Composed of one or more `FeatureCharacteristic` objects [S1, unofficial].
4.  **Feature Characteristic:** The lowest layer, storing:
    *   `attribute`: The name of the data attribute (e.g., "pressure") [S1, unofficial].
    *   `range`: A smooth, brushed range (1D for histograms, 2D for scatterplots) [S1, unofficial].
    *   `combination`: Logical operators allowing characteristics to be combined (e.g., AND, OR) [S1, unofficial].
    *   *Note:* Spatial dimensions can be included as an additional attribute to refine definitions based on spatial position [S1, unofficial].

**Steps:**
1.  **Initialize Linked Views:** Create multiple linked views (histograms, scatterplots) depicting relationships between data attributes [S1, unofficial].
2.  **Brush Selection:** The user performs a smooth selection (brushing) on a scatterplot or histogram [S1, unofficial]. For example, selecting a region in a plot of absolute pressure vs. velocity [S1, unofficial].
3.  **Propagate Selection:** The selection immediately updates the visual appearance of all linked views, highlighting the corresponding ranges in other attributes (e.g., viscosity vs. turbulence kinetic energy) [S1, unofficial].
4.  **Iterate:** Repeat steps 2-3 in one or two-dimensional domains until the desired structure is found [S1, unofficial].
5.  **Refine:** Use sliders or numerical input to refine approximate selections within the allowed range of data attributes [S1, unofficial].
6.  **Generate FDL:** The system compiles the interactive selections into the FDL structure [S1, unofficial].
7.  **Compute DOI:** Calculate the smooth DOI function based on the FDL [S1, unofficial].

**Formulas:**
The DOI function $D(p)$ for a sample point $p$ is computed by evaluating the logical combination of all active `FeatureCharacteristic` ranges in the FDL [S1, unofficial].
$$ D(p) \in [0, 1] $$
Where $D(p)=1$ indicates full interest and $D(p)=0$ indicates no interest [S1, unofficial]. The exact mathematical formulation of the "smooth" range and logical combination is not specified in the sources [S1, unofficial].

**Parameters:**
*   **Spatial Dimension:** Optional. If included, it acts as an additional attribute in the FDL characteristics [S1, unofficial].
*   **Brush Smoothness:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class FeatureCharacteristic:
    def __init__(self, attribute, range_function, logical_op):
        self.attribute = attribute
        self.range_function = range_function # Returns float in [0,1] for a value
        self.logical_op = logical_op # e.g., 'AND', 'OR'

class Feature:
    def __init__(self):
        self.characteristics = [] # List of FeatureCharacteristic

    def evaluate(self, data_point):
        # Combine characteristics using logical operations
        # Note: Exact combination logic (e.g., t-norms) is not specified
        result = 1.0
        for char in self.characteristics:
            val = char.range_function(data_point[char.attribute])
            if char.logical_op == 'AND':
                result = min(result, val) # Assumption for smooth AND
            elif char.logical_op == 'OR':
                result = max(result, val) # Assumption for smooth OR
        return result

class FeatureSet:
    def __init__(self):
        self.features = [] # List of Feature

    def evaluate(self, data_point):
        # Combine features in the set
        # Note: Exact combination logic is not specified
        max_interest = 0.0
        for feature in self.features:
            interest = feature.evaluate(data_point)
            if interest > max_interest:
                max_interest = interest
        return max_interest

class FDL:
    def __init__(self, filename):
        self.filename = filename
        self.feature_sets = [] # List of FeatureSet

    def get_doi(self, data_point):
        # Compute DOI for a single point across all sets
        # Note: Logic for combining sets is not specified
        doi = 0.0
        for fs in self.feature_sets:
            val = fs.evaluate(data_point)
            if val > doi:
                doi = val
        return doi

def interactive_brushing_session(data, views):
    """
    Main loop for the interaction.
    """
    fdl = FDL(data.filename)
    
    while not user_finished():
        # 1. User brushes in one view (e.g., scatterplot of attr A vs B)
        selection = get_user_brush(views)
        
        # 2. Update FDL with new characteristic
        # Note: Implementation of 'smooth range' creation is not specified
        new_char = create_smooth_characteristic(selection.attribute, selection.range)
        
        # 3. Add to current feature set
        current_feature_set.add_feature(new_char)
        
        # 4. Propagate to linked views
        update_linked_views(views, fdl, data)
        
        # 5. Refine if needed
        if user_requests_refinement():
            refine_selection(new_char)
            
    return fdl
```

**Edge Cases and Pitfalls:**
*   **Precision:** The "smooth" nature of the DOI requires careful handling of boundary conditions to avoid abrupt transitions [S1, unofficial].
*   **Degenerate Input:** If no data points fall within the brushed range, the DOI for those points is 0 [S1, unofficial].
*   **High Dimensionality:** As the number of attributes increases, the complexity of linking and interpreting multiple views grows [S1, unofficial].

## Subtopics

### Lock Tool
A sibling method for selection and manipulation. [Lock Tool](13353-lock-tool.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Intuitive and straightforward for large, high-dimensional data; allows identification of complex features through composite brushes [S1, unofficial].
*   **Limitations:** Requires careful management of linked views; refinement may be needed for approximate selections [S1, unofficial].
*   **When to Use:** When users need to interactively define complex, multi-attribute features in simulation data [S1, unofficial].
*   **When Not to Use:** When real-time performance is critical and the overhead of maintaining linked views and smooth DOI calculations is too high [S1, unofficial].

## Background
The method was introduced in the paper "Feature-based Volume Rendering of Simulation Data" by M. Mlejnek in 2003 [S1, unofficial]. It is part of a framework for visualizing computational flow simulation data, leveraging the concept of Degree of Interest (DOI) to guide volume rendering [S1, unofficial].

## Sources
[S1] Feature-based Volume Rendering of Simulation Data, https://old.cescg.org/CESCG-2003/MMlejnek/index.html, fetched 2026-10-27 (unofficial)
