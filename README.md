It's a fork of JoyShockLibrary 3.0.2. from r57zone https://github.com/r57zone/JoyShockLibrary. <br>
Fork was created for JCADvance, but it may also work with DSadvance. Look for changes in the code using the special characters: //@

JoyShockLibrary Modification Summary (JCAdvance Fork)

1. Gyro Axis Overwrite Bug Fix <br>
Code: //@301 <br>
Files modified: JoyShock.cpp <br>
Function modified: get_and_flush_cumulative_gyro() <br>
Description: Resolved a critical typo where gyroY and gyroZ assignments were mistakenly written to gyroX when num_cumulative_gyro_samples == 0. This bug previously corrupted the X-axis and left the Y/Z axes unassigned, leading to severe camera stuttering and axis drift on frames where no new IMU samples arrived.

2. Double-Transformation Gyro Space Bug Fix <br>
Code: //@201 <br>
Files modified: JoyShockLibrary.cpp <br>
Function modified: pollIndividualLoop() <br>
Description: Fixed a logical flaw where already-transformed spatial gyro values (World/Player Space) were pushed into the cumulative gyro buffer. This caused JSL to apply the spatial rotation projection matrix twice during the final accumulation flush, corrupting the coordinate system when gyroSpace = 1 or 2 was enabled [2]. Pushing raw, untransformed calibrated IMU data instead allows the spatial math to execute exactly once, enabling proper wrist-tilt compensation.

3. Data Race & Input Flickering Prevention (Double-Buffering) <br>
Code: //@601 <br>
Files modified: InputHelpers.cpp <br>
Function modified: handle_input() <br>
Description: Implemented a thread-safe double-buffering pattern to prevent state tearing. Previously, the background polling thread directly cleared and gradually rebuilt active button/stick registers on jc->simple_state and jc->touch_state, which occasionally coincided with main-thread reads. Now, all calculations are safely parsed into local stack structures (local_simple_state, local_touch_state) and copied over to global state atomically under the protection of modifying_lock at the very end of the polling cycle.

4. Thread-Safe State Readers & IMU Function Overloads <br>
Code: //@202, //@302
Files modified: JoyShockLibrary.cpp, JoyShock.cpp,  <br>
Functions modified/added: get_transformed_imu_state(), JslGetSimpleState(), JslGetIMUState(), JslGetGyro*(), JslGetAccel*() <br>
Description: Overloaded the get_transformed_imu_state() method to support parameter-free invocation, allowing thread-safe, lock-guarded access to internal IMU structures. Protected main-thread reading in JslGetSimpleState() with modifying_lock and updated all single-axis gyro and accelerometer getters to retrieve state via the thread-safe copy generator. This eliminates any remaining data races between JSL's background polling loop and the host application's rendering frame rates.

5. Experemental case 3 (hybrid) for Joy-cons  <br>
Code: //@203, //@303, //@401  <br>
Files modified: JoyShockLibrary.cpp, JoyShock.cpp, GamepadMotion.hpp  <br>
Description: In horizontal grip (R and ZR pointing at the screen) uses case 1, as the vertical angle increases, a gradual transition to case 0.

6. //@305 JoyShock.cpp - Fine-tuning calibration settings for JOY-CON

7. OSD <br>
Code:  //@204, //@205, //@304, //@501, //@602 <br>
Files modified: JoyShockLibrary.cpp, JoyShock.cpp, JoyShockLibrary.h, InputHelpers.cpp <br>
Description: Modification and bugfixes (//@602) of the library to display information on the OSD

8. Auto-calibration fix <br>
Code: //@402 <br>
Files modified: GamepadMotion.hpp  <br>
Description: The original algorithm permanently locked the noise threshold to the absolute historical minimum. As the controller heated up during play, its natural sensor noise slightly increased, making it physically impossible to meet the old, overly strict threshold again. <br>
The Solution: Implemented an adaptive threshold relaxation mechanism in GamepadMotion.hpp. If the controller fails the strict stillness check, but the current gyro noise is under a safe physical limit (< 0.5 deg/s, meaning the controller is definitely on a desk, not in hands), the stuck threshold smoothly interpolates (relaxes) towards the current noise level. This ensures auto-calibration never "dies" and dynamically adapts to the current thermal and noise state of the IMU without triggering false calibrations in your hands.

9. Auto-calibration settings to config.ini (for Nintendo only) <br>
Code: //@206, //@502 <br>
Files modified: JoyShockLibrary.cpp, JoyShockLibrary.h <br>
Description: added MaxStillnessError, MinStillnessCollectionTime, MinStillnessCorrectionTime, StillnessCalibrationEaseInTime to config.ini [Joy-Con] section

## Builings
How to see in building section: https://github.com/fttlov/JCAdvance
