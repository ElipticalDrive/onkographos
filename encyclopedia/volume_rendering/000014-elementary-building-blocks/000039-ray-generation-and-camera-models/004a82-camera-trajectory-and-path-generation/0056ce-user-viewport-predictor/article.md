# User Viewport Predictor

Also called: Viewport trajectory prediction

## Overview
The User Viewport Predictor estimates future camera poses (6DoF) from recent interaction history to enable proactive resource allocation in volume rendering and 3D scene delivery [S1]. It differs from synthetic or spline-based trajectory generators by relying on empirical user data rather than predefined geometric paths [S1]. It serves as a lightweight, short-term forecasting module that can be replaced by more complex models like deep learning predictors [S1].

## How it works
The method uses linear regression to predict the next $N$ frames based on a sliding window of recent observations [S1].

**Inputs and Outputs**
*   **Input**: A sequence of 6DoF viewport poses $\mathbf{P}_t = [x_t, y_t, z_t, \alpha_t, \beta_t, \gamma_t]$ for $t$ in the history window $hw$ [S1].
*   **Output**: Predicted 6DoF poses for the prediction window $pw$ [S1].

**Data Structures**
*   **History Window ($hw$)**: A buffer storing recent viewport samples [S1].
*   **Prediction Window ($pw$)**: The temporal range into the future to predict [S1].
*   **Linear Model**: For each of the 6 degrees of freedom, a separate linear regression model $y = mx + c$ is maintained [S1].

**Parameters**
*   $hw = 0.5$ seconds [S1].
*   $pw = 1$ second [S1].

**Steps**
1.  **Re-center Angular Data**: Before fitting, handle the cyclical nature of rotation angles ($\alpha, \beta, \gamma$) [S1]. Set the angle of the first point in $hw$ as $0$ [S1]. Calculate subsequent angles relative to this reference, increasing clockwise and decreasing counter-clockwise, resulting in a range of $[-180, 180]$ [S1].
2.  **Fit Linear Model**: For each of the 6 features independently, treat the feature value as the dependent variable $y_i$ and its relative order in the sequence compared to the first point in $hw$ as the independent variable $x_i$ [S1]. Compute the linear regression coefficients (slope $m$ and intercept $c$) for the set $\{(x_i, y_i)\}$ [S1].
3.  **Forecast**: For each future time step $k$ in $pw$, calculate the relative time index $x_{pred}$ from the start of $hw$ and predict the feature value $y_{pred} = m \cdot x_{pred} + c$ [S1].
4.  **Update**: Continuously slide the $hw$ window forward as new viewport data arrives and re-fit the models [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class UserViewportPredictor:
    def __init__(self, history_window_sec=0.5, pred_window_sec=1.0, fps=36):
        self.history_window_sec = history_window_sec
        self.pred_window_sec = pred_window_sec
        self.fps = fps
        self.history_len = int(history_window_sec * fps)
        self.pred_len = int(pred_window_sec * fps)
        
        # Buffer for 6DoF: [x, y, z, alpha, beta, gamma]
        self.history_buffer = [None] * self.history_len
        self.model_params = [[0.0, 0.0] for _ in range(6)] # [slope, intercept] per DOF
        self.filled_count = 0

    def _recenter_angles(self, angles):
        """Re-center angular data relative to the first point in the window."""
        if not angles or len(angles) == 0:
            return angles
        first = angles[0]
        # Relative angles, range [-180, 180] implied by source
        return [math.degrees(math.atan2(math.sin(math.radians(a - first)), 
                                        math.cos(math.radians(a - first)))) 
                for a in angles]

    def _linear_regression(self, x_vals, y_vals):
        """Compute slope (m) and intercept (c) for y = mx + c."""
        n = len(x_vals)
        if n < 2:
            return 0.0, 0.0
        sum_x = sum(x_vals)
        sum_y = sum(y_vals)
        sum_xy = sum(x * y for x, y in zip(x_vals, y_vals))
        sum_x2 = sum(x * x for x in x_vals)
        denom = n * sum_x2 - sum_x * sum_x
        if denom == 0:
            return 0.0, sum_y / n
        m = (n * sum_xy - sum_x * sum_y) / denom
        c = (sum_y - m * sum_x) / n
        return m, c

    def update(self, new_pose):
        """Add a new viewport pose and re-fit models."""
        # Shift buffer and append new pose
        self.history_buffer.pop(0)
        self.history_buffer.append(new_pose)
        self.filled_count = min(self.filled_count + 1, self.history_len)
        
        # Fit models for each DOF
        for dof in range(6):
            # Extract valid samples
            samples = [self.history_buffer[i][dof] for i in range(self.filled_count)]
            x_vals = list(range(self.filled_count))
            
            # Handle angles (dofs 3, 4, 5)
            if dof >= 3:
                samples = self._recenter_angles(samples)
            
            m, c = self._linear_regression(x_vals, samples)
            self.model_params[dof] = [m, c]

    def predict(self):
        """Predict future viewports for the prediction window."""
        if self.filled_count < 2:
            return None
            
        # Relative time indices for prediction continue from history
        start_x = self.filled_count
        x_pred = list(range(start_x, start_x + self.pred_len))
        
        predictions = []
        for k in range(self.pred_len):
            row = []
            for dof in range(6):
                m, c = self.model_params[dof]
                y_pred = m * x_pred[k] + c
                row.append(y_pred)
            predictions.append(row)
        return predictions
```

**Edge Cases and Pitfalls**
*   **Angular Discontinuity**: The cyclical nature of angles ($0^\circ \equiv 360^\circ$) can cause large errors in linear regression if not handled [S1]. The method requires re-centering relative to the first point in the history window [S1].
*   **Non-Linear Motion**: Linear regression assumes constant velocity/rotation [G]. Sudden user movements may lead to prediction errors [G].

## Subtopics

### [Engagement Estimation](15829-engagement-estimation.md)
A module that processes user interaction data to evaluate the type, parameters, speed, and duration of engagement, potentially predicting consecutive viewpoints or regions of interest. It serves as a bridge between raw input events and the visualization system's response, specifically feeding into the User Viewport Predictor to inform future camera trajectory decisions. It is distinguished from the predictor itself by its role in quantifying the quality and intent of the current interaction (e.g., zooming vs. panning) rather than just fitting a linear regression to historical pose data.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per frame for fitting, as the number of features (6) and history window size (fixed at 0.5 s) are constant [G].
*   **Memory Complexity**: $O(hw \cdot 6)$, where $hw$ is the history window size in samples [G].
*   **Performance**: Simple and fast, suitable for real-time systems [S1].
*   **Accuracy**: Reasonable accuracy for short-term forecasting [S1].
*   **Limitations**: Assumes linear motion; may fail during rapid or non-linear user interactions [G].

## Background
The method was introduced in the context of L3GS (Layered 3D Gaussian Splats for Efficient 3D Scene Delivery) [S1]. It was chosen for its simplicity, speed, and reasonable accuracy in short-term forecasting, similar to its use in other viewport-adaptive systems [S1]. The modular design allows for future replacement with more sophisticated models like deep learning predictors [S1].

## Sources
[S1] L3GS: Layered 3D Gaussian Splats for Efficient 3D Scene Delivery, https://doi.org/10.1145/3680207.3723485, fetched 2026-10-27
