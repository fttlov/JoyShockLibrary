It's a fork of JoyShockLibrary 3.0.2. from r57zone https://github.com/r57zone/JoyShockLibrary. 
Fork was created for JCADvance, but it may also work with DSadvance. Look for changes in the code using the special characters: //@

JoyShockLibrary Modification Summary (JCAdvance Fork)
1. Gyro Axis Overwrite Bug Fix
Files modified: JoyShock.cpp
Function modified: get_and_flush_cumulative_gyro()
Description: Resolved a critical typo where gyroY and gyroZ assignments were mistakenly written to gyroX when num_cumulative_gyro_samples == 0. This bug previously corrupted the X-axis and left the Y/Z axes unassigned, leading to severe camera stuttering and axis drift on frames where no new IMU samples arrived.

2. Double-Transformation Gyro Space Bug Fix
Files modified: JoyShockLibrary.cpp
Function modified: pollIndividualLoop()
Description: Fixed a logical flaw where already-transformed spatial gyro values (World/Player Space) were pushed into the cumulative gyro buffer. This caused JSL to apply the spatial rotation projection matrix twice during the final accumulation flush, corrupting the coordinate system when gyroSpace = 1 or 2 was enabled [2]. Pushing raw, untransformed calibrated IMU data instead allows the spatial math to execute exactly once, enabling proper wrist-tilt compensation.

3. Data Race & Input Flickering Prevention (Double-Buffering)
Files modified: InputHelpers.cpp
Function modified: handle_input()
Description: Implemented a thread-safe double-buffering pattern to prevent state tearing. Previously, the background polling thread directly cleared and gradually rebuilt active button/stick registers on jc->simple_state and jc->touch_state, which occasionally coincided with main-thread reads. Now, all calculations are safely parsed into local stack structures (local_simple_state, local_touch_state) and copied over to global state atomically under the protection of modifying_lock at the very end of the polling cycle.

4. Thread-Safe State Readers & IMU Function Overloads
Files modified: JoyShock.cpp, JoyShockLibrary.cpp
Functions modified/added: get_transformed_imu_state(), JslGetSimpleState(), JslGetIMUState(), JslGetGyro*(), JslGetAccel*()
Description: Overloaded the get_transformed_imu_state() method to support parameter-free invocation, allowing thread-safe, lock-guarded access to internal IMU structures. Protected main-thread reading in JslGetSimpleState() with modifying_lock and updated all single-axis gyro and accelerometer getters to retrieve state via the thread-safe copy generator. This eliminates any remaining data races between JSL's background polling loop and the host application's rendering frame rates.

5. Experemental case 3 (hybrid) for Joy-cons. In horizontal grip (R and ZR pointing at the screen) uses case 1, as the vertical angle increases, a gradual transition to case 0.

6. Debug to see the actual polling rate of your devices. Uncomment in void get_and_flush_cumulative_gyro(float& gyroX, float& gyroY, float& gyroZ for use it.

## Builings
How to see in building section: https://github.com/fttlov/JCAdvance
