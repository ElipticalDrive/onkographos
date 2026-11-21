# Distance-Controlled Depth Pointing

Also called: Ultrasonic distance-based pointing

## Overview
Distance-Controlled Depth Pointing is an interaction method for swept volumetric displays where the depth of a laser dot along the intersection line is determined by the physical distance between the pointer device and the display surface [S1]. This creates the illusion that the dot appears at a fixed distance from the pointer, effectively implementing a "lock ray" technique where the marker's position is controlled by inputs from the pointer device rather than a phase offset [S1]. It serves as an alternative to Phase-Controlled Laser Pointing for Swept Displays, which uses an analog joystick to adjust phase [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Physical distance from the pointer to the display center, measured via an ultrasonic sensor.
*   **Output:** Timing signal to trigger the laser diode at the correct phase of the display's sweep cycle to render the dot at the calculated depth.

**Data Structures:**
*   **Sensor Configuration:**
    *   Sensor Model: HC-SR04 ultrasonic range sensor [S1].
    *   Sampling Rate: 150 samples per second at 1 meter distance [S1].
    *   Default Field of View Angle: 15 degrees [S1].
    *   Modified Field of View Angle: 25 degrees (achieved by increasing pulse voltage $V_{pp}$ from 5V to 30V) [S1].
*   **Microcontroller State:**
    *   Reference Distance: A stored value representing the distance from the center of the display to the pointer at the moment of reset [S1].
    *   Current Distance: The latest corrected reading from the sensor [S1].
    *   Predicted Difuser Position: A calculated value based on the sinusoidal oscillation model of the display's difuser [S1].

**Numbered Steps:**
1.  **Measure Raw Distance:** The ultrasonic sensor emits a sound pulse and measures the time of flight ($t_{tof}$) for the echo to return [S1].
2.  **Correct for Time of Flight:** The raw distance reading is corrected to account for the time of flight of sound to ensure accuracy [S1].
3.  **Reset Reference (Optional):** If the user presses the reset button, the current corrected distance is saved as the new reference distance from the center of the display to the pointer [S1].
4.  **Predict Difuser Position:** The system predicts the position of the display's difuser assuming a sinusoidal oscillation. This prediction is used to determine the precise timing required to activate the laser [S1].
5.  **Trigger Laser:** The laser diode is triggered at the calculated phase to render the dot at the depth corresponding to the pointer's distance [S1].

**Formulas:**
*   **Distance Correction:** The sources state that distances are corrected according to the time of flight of sound, but do not provide the specific mathematical formula for this correction [S1].
*   **Difuser Position Prediction:** The position is predicted assuming a sinusoidal oscillation of the difuser. The specific equation for this sinusoidal model is not provided in the sources [S1].

**Parameters:**
*   **Laser:** 1 mW red laser diode [S1].
*   **Driver:** 2N2222 transistor [S1].
*   **Microcontroller:** Arduino Nano [S1].
*   **Power:** 9V rechargeable battery [S1].
*   **Display Frequency:** 30 Hz (default) [S1].
*   **Laser Pulse Voltage ($V_{pp}$):** 30 V (modified) or 5 V (default) [S1].

**Edge Cases and Pitfalls:**
*   **Sensor Exposure:** The difuser must be directly exposed to the ultrasonic sensor. A protective dome cannot be used because ultrasound requires direct exposure to propagate effectively [S1].
*   **LIDAR Rejection:** LIDAR sensors were explored but rejected due to low sampling rates (below 30 samples per second) and unreliability caused by the translucency of the difuser [S1].
*   **Drift:** With a default frequency of 30 Hz and an Arduino Nano (Timer 1 at 16 MHz resolution), the laser drifts only 5 mm per minute in a Voxon VX1 display [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SPEED_OF_SOUND = 343.0  # m/s, standard value [G]
DISPLAY_FREQ_HZ = 30.0  # Hz [S1]
SAMPLE_RATE_HZ = 150.0  # Hz [S1]

# State
reference_distance_m = 0.0
current_distance_m = 0.0

def measure_raw_distance():
    """
    Emits ultrasonic pulse and measures time of flight.
    Returns raw time in seconds.
    """
    # Implementation depends on HC-SR04 hardware interface [S1]
    # Trigger pin set high for 10us, wait for echo high, measure duration
    pass

def correct_distance(raw_time_s):
    """
    Corrects raw time of flight to distance.
    Formula: distance = (time * speed_of_sound) / 2
    """
    return (raw_time_s * SPEED_OF_SOUND) / 2.0

def predict_difuser_position(time_s):
    """
    Predicts difuser position assuming sinusoidal oscillation.
    Specific formula not provided in sources [S1].
    """
    # Placeholder for sinusoidal model [G]
    # position = A * sin(2 * pi * DISPLAY_FREQ_HZ * time_s + phase)
    pass

def reset_reference():
    """
    Sets the current corrected distance as the reference distance
    from the center of the display to the pointer [S1].
    """
    global reference_distance_m
    reference_distance_m = current_distance_m

def trigger_laser():
    """
    Calculates laser trigger timing based on current distance
    and difuser position prediction [S1].
    """
    global current_distance_m
    
    # 1. Measure and correct distance [S1]
    raw_time = measure_raw_distance()
    current_distance_m = correct_distance(raw_time)
    
    # 2. Predict difuser position [S1]
    # This step requires synchronization with the display's sweep cycle
    difuser_pos = predict_difuser_position(get_current_time())
    
    # 3. Calculate phase offset based on distance
    # The depth is determined by the distance between pointer and display [S1]
    # Specific mapping from distance to phase is not provided [S1]
    
    # 4. Trigger laser
    # Laser switches on at configured frequency matching display oscillation [S1]
    # Trigger logic depends on the specific microcontroller timer setup [S1]
    pass

def main_loop():
    while True:
        if button_pressed():
            reset_reference()
        trigger_laser()
        sleep(1.0 / SAMPLE_RATE_HZ)
```

## Operating characteristics
*   **Accuracy:** The method provides a fixed-distance illusion relative to the pointer. Drift is minimal (5 mm per minute) when using a 30 Hz display and an Arduino Nano [S1].
*   **Limitations:** The method requires direct line-of-sight between the sensor and the difuser, preventing the use of protective domes [S1]. It is sensitive to the translucency of the difuser, which caused unreliability with LIDAR alternatives [S1].
*   **Hardware Constraints:** Relies on the HC-SR04 sensor, which has a maximum angle of 15 degrees (modifiable to 25 degrees with higher voltage) [S1].

## Background
This method was introduced in the paper "PointerVol: A Laser Pointer for Swept Volumetric Displays" by Fernández et al., presented at UIST 2024 [S1]. It is part of the development of laser-based 3-D volumetric displays using rotating helix geometry and AO scanners. The technique is equivalent to the "lock ray" technique previously shown to yield good results for volumetric displays [S1].

## Sources
[S1] PointerVol: A Laser Pointer for Swept Volumetric Displays, https://doi.org/10.1145/3654777.3676432, fetched 2026-10-30
