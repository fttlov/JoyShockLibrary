It's a fork of JoyShockLibrary 3.0.2. from r57zone https://github.com/r57zone/JoyShockLibrary. <br>
Fork was created for JCADvance, but it may also work with DSadvance. Look for changes in the code using the special characters: //@

### JoyShockLibrary Modification Summary (JCAdvance Fork)

1. #### Gyro Axis Overwrite Bug Fix <br>
**Code:** //@301 <br>
**Files modified:** JoyShock.cpp <br>
**Function modified:** get_and_flush_cumulative_gyro() <br>
**Description:** Resolved a critical typo where gyroY and gyroZ assignments were mistakenly written to gyroX when num_cumulative_gyro_samples == 0. This bug previously corrupted the X-axis and left the Y/Z axes unassigned, leading to severe camera stuttering and axis drift on frames where no new IMU samples arrived.

2. #### Double-Transformation Gyro Space Bug Fix <br>
**Code:** //@201 <br>
**Files modified:** JoyShockLibrary.cpp <br>
**Function modified:** pollIndividualLoop() <br>
**Description:** Fixed a logical flaw where already-transformed spatial gyro values (World/Player Space) were pushed into the cumulative gyro buffer. This caused JSL to apply the spatial rotation projection matrix twice during the final accumulation flush, corrupting the coordinate system when gyroSpace = 1 or 2 was enabled [2]. Pushing raw, untransformed calibrated IMU data instead allows the spatial math to execute exactly once, enabling proper wrist-tilt compensation.

3. #### Data Race & Input Flickering Prevention (Double-Buffering) <br>
**Code:** //@601 <br>
**Files modified:** InputHelpers.cpp <br>
**Function modified:** handle_input() <br>
**Description:** Implemented a thread-safe double-buffering pattern to prevent state tearing. Previously, the background polling thread directly cleared and gradually rebuilt active button/stick registers on jc->simple_state and jc->touch_state, which occasionally coincided with main-thread reads. Now, all calculations are safely parsed into local stack structures (local_simple_state, local_touch_state) and copied over to global state atomically under the protection of modifying_lock at the very end of the polling cycle.

4. #### Thread-Safe State Readers & IMU Function Overloads <br>
**Code:** //@202, //@302
**Files modified:** JoyShockLibrary.cpp, JoyShock.cpp,  <br>
**Functions modified/added:** get_transformed_imu_state(), JslGetSimpleState(), JslGetIMUState(), JslGetGyro*(), JslGetAccel*() <br>
**Description:** Overloaded the get_transformed_imu_state() method to support parameter-free invocation, allowing thread-safe, lock-guarded access to internal IMU structures. Protected main-thread reading in JslGetSimpleState() with modifying_lock and updated all single-axis gyro and accelerometer getters to retrieve state via the thread-safe copy generator. This eliminates any remaining data races between JSL's background polling loop and the host application's rendering frame rates.

5. #### New case 3 (Planar Space) for Joy-cons  <br>
**Code:** //@203, //@303, //@401  <br>
**Files modified:** JoyShockLibrary.cpp, JoyShock.cpp, GamepadMotion.hpp  <br>
**Description:** Modified World Gyro Space: minimal interference with the control of the X-axis when rotating the hand clockwise and anti-clockwise during a vertical grip
Added "Planar Space" (Case 3) for gyro gaming. This mode completely filters out the local Z-axis (roll/"screwdriver" effect) interference to gyro mouse/stick X axis iv vertical grip . It calculates pure 2D planar pitch and yaw, utilizing a squared gravity vector compensation to maintain 1:1 horizontal sensitivity even at high controller angles (ZR pointing more to the ceiling). Perfect for non-horizontal Joy-Con grips. <br>
**Update in 3.8":** The limits on wrist rotation (screwdriver) angles greater than -90 90 degrees have been relaxed for horizontal grip. Vertical (horizon) grip limits still there


6. ~~//@305 JoyShock.cpp - Fine-tuning calibration settings for JOY-CON~~

7. #### OSD <br>
**Code:**  //@204, //@205, //@208, //@304, //@403 //@501, //@503, //@602 <br>
**Files modified:** JoyShockLibrary.cpp, JoyShock.cpp, GamepadMotion.hpp, JoyShockLibrary.h, InputHelpers.cpp <br>
**Description:** Modification and bugfixes (//@602) of the library to display information on the OSD

8. #### Auto-calibration fix <br>
**Code:** //@402 <br>
**Files modified:** GamepadMotion.hpp  <br>
Description: ~~The original algorithm permanently locked the noise threshold to the absolute historical minimum. As the controller heated up during play, its natural sensor noise slightly increased, making it physically impossible to meet the old, overly strict threshold again. <br>
The Solution: Implemented an adaptive threshold relaxation mechanism in GamepadMotion.hpp. If the controller fails the strict stillness check, but the current gyro noise is under a safe physical limit (< 0.5 deg/s, meaning the controller is definitely on a desk, not in hands), the stuck threshold smoothly interpolates (relaxes) towards the current noise level. This ensures auto-calibration never "dies" and dynamically adapts to the current thermal and noise state of the IMU without triggering false calibrations in your hands.
Fixes in JCAdvance 3.5: the sluggish 10-15 second delay in gyro auto-calibration recovery. The relaxation rate now correctly uses the accumulated collection time instead of frame delta, allowing near-instant calibration recovery when placing the controller on a desk.~~
**Updated after:** //@
**Description:** Overhauled the auto-calibration algorithm to seamlessly accommodate the wider raw noise floor introduced by **True 200Hz IMU Unlocking** and long-term (30+ min) sensor thermal drift. Replaced rigid gyro thresholds with an ironclad accelerometer table-detector (`accelDelta < 0.01G`). When resting on a solid surface, the system rapidly adapts the `MinDeltaGyro` noise baseline within ~1 second, eliminating calibration lockouts over extended play sessions while guaranteeing 100% immunity to false in-hand calibrations.

9. #### Auto-calibration settings to config.ini (for Nintendo only) <br>
**Code:** //@206, //@502 <br>
**Files modified:** JoyShockLibrary.cpp, JoyShockLibrary.h <br>
**Description:** added MaxStillnessError, MinStillnessCollectionTime, MinStillnessCorrectionTime, StillnessCalibrationEaseInTime, GravityShakinessMin, GravityShakinessMax, GravityStillSpeed, GravityShakySpeed to config.ini [Joycons] section

10. #### Accelerometer Calibration & Axis Mapping <br>
**Code:** //@604 <br>
**Files modified:** `InputHelpers.cpp` <br>
**Description:** Fixed a critical math bug in how JSL handles accelerometer calibration data from the Nintendo SPI Flash memory. The original code read the zero-offset calibration (sensor_cal[0]) but failed to subtract it from the raw sensor data, only using it to slightly alter the multiplier coefficient. It also contained a typo that applied the X-axis calibration to the physical Y-axis. <br>
Because of this, hardware-calibrated controllers still suffered from severe cross-talk (diagonal drifting) in "World Space" mode. The fix corrects the math formula to (Raw - Zero_Offset) * Coefficient and properly maps the physical axes. Now, JSL flawlessly respects the hardware accelerometer calibration recorded via the Nintendo Switch console or hardware shortcuts. 

11. #### Software accelerometer recalibration (by hotkey) <br>
**Code:** //@207 <br>
**Files modified:** `JoyShockLibrary.cpp` <br>
**Description:** If you cannot perform a hardware calibration of the accelerometer, use software calibration (hotkey) for current session.

12. #### Precise Factory SPI Sensor Calibration & Scaling <br>
**Code:** //@306, 604 <br>
**Files modified:** `JoyShock.cpp`, `InputHelpers.cpp` <br>
**Description:** Replaced legacy hardcoded IMU scaling constants (which blindly assumed `16384` for accelerometer ±4G range and `13371` for gyroscope sensitivity for all controllers) with unique, per-chip factory calibration multipliers read dynamically from the controller's SPI flash memory (`0x6020` factory / `0x8026` user blocks). Corrected the fundamental order of operations to `(Raw - Offset) * UniqueMultiplier` across proper physical axes (Y, X, Z), eliminating scale distortion, fixing skewed gravity vectors, and ensuring 100% physically accurate angular velocity calculations.

13. #### True 200Hz IMU Unlocking & Sequential Sub-Packet Processing  <br>
**Code:** //@604 <br>
**Files modified:** `InputHelpers.cpp` <br>
**Description:** Nintendo Switch controllers sample their IMU at 200Hz (every 5ms) and pack 3 discrete readings into a single Bluetooth HID report sent every 15ms (~66.6Hz). Completely overhauled the input pipeline by removing the legacy code's 3-sample averaging filter, which previously acted as an unintended low-pass filter—blurring sharp flicks and adding artificial input lag. The new pipeline unrolls and processes all 3 micro-samples sequentially with a divided `deltaTime`, tripling motion tracking resolution, maximizing flick-shot responsiveness, and unlocking true 200Hz+ sensor fidelity without increasing Bluetooth bandwidth.

### 14. Modified World Space (//@404) 
- **Description**: Standard JSL World Space pitch (vertical / X-axis) calculation suffers from critical mathematical flaws and cross-axis corruption:<br>
  1. **Pitch Cross-Talk & Arc Distortion**: Original JSL pitch includes a `-gravZ * gravX * gyroZ` term. At non-zero pitch angles, pure wrist rolls (screwdriver gesture) caused the cursor to sag in a curved downward arc, and diagonal swipes warped into unpredictable S-curves, leading to vertical height overshoot. <br>
  2. **Roll = 90° Freeze**: Holding the controller on its side causes the projected `pitchAxis` vector to collapse to `(0,0,0)`, completely freezing vertical cursor movement. Furthermore, original `sideReduction` ignores X-axis gravity (`gravX`), evaluating to `0.0` and creating a secondary input lock. <br>
  3. **Roll > 90° Inversion**: Rolling the controller past 90° inverts pitch projection polarity, moving the cursor down when pitching the hand up. <br>
  4. **Zenith Micro-Freeze (Pitch = 90°)**: Pointing straight at the ceiling causes 2D gravity projections to collapse, leading to sub-frame dead zones when crossing the zenith. <br>
- **Fix**: Replaced JSL's pitch projection with a robust 3D-aware Virtual Un-Roll and Zenith Cross-Fade algorithm: <br>
  - **Isolated Pitch Tracking (`unrolledGyroX`)**: Excludes `gyroZ` (wrist roll) from the vertical axis entirely. Restores 100% pitch responsiveness at Roll = 90°, prevents pitch inversion at Roll > 90°, ensures wrist rolls at Pitch > 0 move the cursor in a perfectly flat horizontal line (no downward arc sag), and guarantees clean, linear diagonal swipes. <br>
  - **Zenith Cross-Fade (`zenithBlend`)**: Smoothly cross-fades the un-rolled gyro into raw `gyroX` near Zenith based on `abs(gravZ)`, completely eliminating micro-freezes and 0/0 division dead zones when pointing straight up or down. <br>
  - **Zenith Polarity Guard (`effectiveGravY`)**: Preserves natural pitch direction when crossing the zenith over the shoulder (`gravY > 0`). <br>
  - **3D Side Reduction (`sideness`)**: Added `sideness = std::abs(gravX)` to `sideReduction`, preventing artificial input muting when holding the controller on its side.

### 15 Connection Handshake & IMU Initialization Loop Fix
Code: @604 <br>
**Files modified:** `InputHelpers.cpp`  <br>
**Description:** Adding zeroSampleCount++ to @604. Resolved a race condition during Bluetooth controller startup where temporary zero-payloads in early sub-packets prematurely flagged `hasIMU = false`. The IMU presence check now evaluates all 3 sub-samples per report, requiring a full zero-payload across the entire packet before declaring the sensor inactive. This completely eliminates infinite `enable_IMU` command spam and reconnection loops during controller connection.

16. #### Angular Velocity Telemetry for OSD <br>
**Code:** //@210, //@504 <br>
**Files modified:** `JoyShockLibrary.cpp`, `JoyShockLibrary.h` <br>
**Functions added:** `JslGetAngularVelocity()` <br>
**Description:** Added an exported API function to retrieve the absolute angular velocity vector length in degrees per second ($\sqrt{\omega_x^2 + \omega_y^2 + \omega_z^2}$) from calibrated gyro data. Designed for real-time motion telemetry in AHK and RTSS OSD overlays without modifying internal accumulator registers.

17. #### DualSense Extended Mode & High Polling Rate Fix (500Hz+) <br>
**Code:** //@211, //@307 <br>
**Files modified:** `JoyShockLibrary.cpp`, `JoyShock.cpp` <br>
**Functions modified/added:** `init_ds5_bt()`, `JslConnectDevices()` <br>
**Description:** Fixed a legacy JSL limitation where DualSense controllers connected over Bluetooth defaulted to simple HID mode (Report `0x01`), hard-capping polling rates to 62.5Hz with disabled motion sensors. Implemented an active initialization handshake via Output Report `0x31` protected by a 32-bit CRC with the required `0xA2` seed. This unlocks native Bluetooth Extended Mode, restoring full 6-axis gyro/accelerometer data and enabling high polling rates (500–650Hz+ depending on the host BT adapter).

18. #### Fast Connection Optimization & Redundant USB Check Removal <br>
**Code:** @212, //@308 <br>
**Files modified:** `JoyShockLibrary.cpp`, `JoyShock.cpp` <br>
**Function modified:** `init_bt()` <br>
**Description:** Restricted the aggressive wired USB handshake (`0x80 0x01`) exclusively to Switch Pro Controllers (`left_right == 3`). Joy-Cons operating over Bluetooth no longer receive unhandled USB handshake packets, eliminating five-second timeout delays, false console error warnings, and connection race conditions on startup.

19. #### Joy-Con Bluetooth Connection & Subcommand Handshake Order <br>
**Code:** //@309 <br>
**Files modified:** `JoyShock.cpp` <br>
**Function modified:** `init_bt()` <br>
**Description:** Resolved an issue where Joy-Con clones (e.g., Mobapad) intermittently booted in a "zombie" state with non-functional motion sensors. Aligned the initialization sequence with the upstream Linux kernel standard (`hid-nintendo.c`): SPI flash calibrations are read first, followed by switching to standard report mode `0x30`, enabling the 6-axis IMU (`subcmd 0x40`), and enabling vibration (`subcmd 0x48`). Added safe 20ms spacing intervals to prevent MCU command buffer saturation and ensure reliable IMU activation on every connection.

20. #### Unified Real-Time Battery Reporting (All Controllers) <br>
**Code:** //@213, //@605 <br>
**Files modified:** `JoyShockLibrary.cpp`, `InputHelpers.cpp` <br>
**Functions modified:** `handle_input()`, `JslGetBattery()` <br>
**Description:** Overhauled battery level reporting across all supported controllers. Added real-time extraction of the native battery byte directly from incoming streaming packets: DualShock 4 (bytes 30/32 for USB/BT) and DualSense (bytes 53/54 for USB/BT). Extended `JslGetBattery()` to return scaled 0–100% capacity for Sony hardware alongside Nintendo devices, eliminating the legacy `-1.0` fallback and providing zero-latency battery telemetry without sending redundant feature report requests.

## Builings
How to see in building section: https://github.com/fttlov/JCAdvance
