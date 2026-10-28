# RoboGrinder Embedded Manual

> A step-by-step guide for doing embedded work on RoboGrinder robots: bringing up a new robot, calibrating motors and the IMU, tuning PID, and keeping the referee system up to date.
> 
> 
> Author: **Rui Ma** · Revised by **Britny Jiang**
> 

**How to use this manual**

- **New members:** read [Safety](#1-safety-read-this-first) and [Tools](#2-tools-and-setup) first. Then follow [Bringing Up a Robot](#3-bringing-up-a-robot-recommended-order) in order.
- **Everyone else:** jump to the component or task you're working on. Each section lists **what you need**, **steps**, and **how to check it worked**.
- Every robot follows the same steps. Where something differs by robot or by motor type, it's called out explicitly.

**Robot types:**

| Robot | Projectile |
| --- | --- |
| Infantry | 17 mm (marble size) |
| Sentry (Shao Bing) | 17 mm (marble size) |
| Hero | 42 mm (golf ball size) |
| Engineer | — |

> 📌 **Rule change for this season:** The new rules **remove the Infantry and add the Engineer robot.** Infantry content stays in this manual for reference, since the Sentry shares most of its code.
> 

> ⚠️ **Line numbers are only a guide.** They shift between code versions. What matters is that you're on the **line with the code shown**. Use VS Code search (Ctrl+F) to find it.
> 

---

## Table of Contents

- [0. Robot Hardware Overview](#0-robot-hardware-overview)
- [1. Safety (Read This First)](#1-safety-read-this-first)
- [2. Tools and Setup](#2-tools-and-setup)
- [3. Bringing Up a Robot: Recommended Order](#3-bringing-up-a-robot-recommended-order)
- [4. DJI (RM) Motors: 3508 / 2006](#4-dji-rm-motors-3508--2006)
- [5. DM 4310 Motors (Yaw & Pitch)](#5-dm-4310-motors-yaw--pitch)
- [6. Gimbal Zero Point and Limits in Code](#6-gimbal-zero-point-and-limits-in-code)
- [7. C-Board Orientation and IMU Calibration](#7-c-board-orientation-and-imu-calibration)
- [8. Chassis Parameters](#8-chassis-parameters)
- [9. Pitch Gravity Compensation](#9-pitch-gravity-compensation)
- [10. PID Tuning (Pitch & Yaw)](#10-pid-tuning-pitch--yaw)
- [11. Feeder (Bullet Feed) Debugging](#11-feeder-bullet-feed-debugging)
- [12. Projectile Initial Velocity](#12-projectile-initial-velocity)
- [13. Operator UI](#13-operator-ui)
- [14. OLED Screen](#14-oled-screen)
- [15. Referee System](#15-referee-system)
- [16. Adding New Hardware (Example: Radar Mount)](#16-adding-new-hardware-example-radar-mount)
- [17. Troubleshooting and Known Issues](#17-troubleshooting-and-known-issues)
- [18. Glossary](#18-glossary)

---

## 0. Robot Hardware Overview

Standard motor and ESC layout (Season 26):

| Subsystem | Motor | ESC / Driver | Count | Control interface |
| --- | --- | --- | --- | --- |
| Chassis | DJI M3508 | C620 | 4 | CAN |
| Yaw | DM 4310 | built-in | 1 | CAN |
| Pitch | DM 4310 | built-in | 1 | CAN |
| Feeder (magazine wheel) | DJI M2006 | C610 | 1 | CAN |
| Friction wheels | DJI M3508 (**no gearbox**) | C620 | 2 | CAN |

> ⚠️ The **C615** ESC is for DJI **Snail** motors (older friction-wheel motors). Don't mix them up. Snail motors were replaced by 3508s because they have no closed-loop control and can't detect stalling.
> 

The main controller is the RoboMaster **Development Board Type C** ("C-board").

---

## 1. Safety (Read This First)

These rules apply to **every** task in this manual.

- [ ]  **Wireless kill switch:** Before debugging any PID loop, or any new robot, have the wiring person connect a **wireless relay/switch**. It lets you cut power to the whole robot if it runs away (this has hurt people before).
- [ ]  **Configuring DM motors:** Disconnect the motor's **power and CAN** from the robot first. Otherwise the external battery can power the whole robot and cause CAN interference.
- [ ]  **Test before installing:** After changing motor IDs, test each motor **once more before it goes on the robot**, so you don't have to take things apart again.
- [ ]  **Know the e-stop:** Know how to power off before you power on.
- [ ]  **New battery:** The new battery shuts down slowly, so an additional switch is required.
- [ ]  **Reset the C-board** after a debugging session. Leaving it in a debug state can cause unexplained bugs.

---

## 2. Tools and Setup

We use **VS Code for every task.** For environment setup, see the team's environment setup doc: `https://github.com/RoboGrinder-at-Virginia-Tech/RG-embed-readme/blob/main/RG-embed-envir-setup.md`.

| Task | Tool |
| --- | --- |
| Editing, building, and flashing code | VS Code |
| **Just flashing code** | **DAPLink** (more convenient) |
| **Tuning motor PID / reading live variables** | **J-Link + Ozone** |
| Breakpoint debugging (e.g., IMU calibration) | VS Code debugger |
| Configuring DM motors (IDs, zero point, PMAX/VMAX/TMAX, firmware) | DM AC-Tools |
| Referee system firmware | RMTool2 |
| Mass and center of mass for gravity compensation | SolidWorks (ask the Mechanical branch) |
| Auto-aim PID waveforms | RQT Plot (with the auto-aim team) |

**Tutorial (VS Code + Ozone + J-Link + CMake):** [MRobot Mini-Class on Bilibili](https://www.bilibili.com/video/BV1AbnYzNEdE/?share_source=copy_web&vd_source=8971dda136e3061f843e58832b8482df)

**Reference docs (Google Drive):** [2024–2025 folder](https://drive.google.com/drive/folders/1-3wWEyQGfoxj2GlW32xoEC2127-q38x5) (includes `BMI088.pdf`)

> 💡 The STM32 supports a **maximum of 5 breakpoints**. If you set more, the code won't run.
> 

### 2.1 Board type (check before flashing)

At the top of `robot_def.h`, **exactly one** board type must be defined:

```c
#define ONE_BOARD        // one board controls the whole robot (current setup)
// #define CHASSIS_BOARD // chassis board
// #define GIMBAL_BOARD  // gimbal board
// #define BALANCE_BOARD // balance chassis, NOT supported, don't use
```

- If more than one is defined, the build fails with `Conflict board definition!`.
- After changing it, **recompile**.

### 2.2 `robot_def.h` at a glance

Most per-robot settings live in `robot_def.h`, including:

- Board type
- Pitch mass / center of mass
- Friction wheel radius and loader bullets per round
- Chassis wheel radius and distance
- IMU rotation matrix
- Referee UI send settings
- Accessories (`SERVO`, `LIDAR_STAND`)

> Floating-point values must end in `.0` or `f`, and unsigned values in `u` (from the file's own comment).
> 

---

## 3. Bringing Up a Robot: Recommended Order

Use this as a checklist when a new robot (or a rebuilt one) arrives. Each step links to its section.

- [ ]  1. Attend the **Mechanical (ME) final review meeting** (see [17.1](#171-me-review-checklist))
- [ ]  2. Set **IDs and direction** for all DJI motors ([§4](#4-dji-rm-motors-3508--2006))
- [ ]  3. Check firmware, then set **IDs, zero point, and PMAX/VMAX/TMAX** for the DM motors ([§5](#5-dm-4310-motors-yaw--pitch))
- [ ]  4. Install motors with the **correct IDs and orientation**, especially the chassis motors!
- [ ]  5. Set the gimbal **zero point and limits** in code ([§6](#6-gimbal-zero-point-and-limits-in-code))
- [ ]  6. Set the **C-board rotation matrix** and **calibrate the IMU** ([§7](#7-c-board-orientation-and-imu-calibration))
- [ ]  7. Enter **chassis parameters** ([§8](#8-chassis-parameters))
- [ ]  8. Calculate and test **pitch gravity compensation** ([§9](#9-pitch-gravity-compensation))
- [ ]  9. **Tune PID** for pitch and yaw, with the kill switch connected ([§10](#10-pid-tuning-pitch--yaw))
- [ ]  10. Set up **feeder** parameters ([§11](#11-feeder-bullet-feed-debugging))
- [ ]  11. Set **projectile speed** ([§12](#12-projectile-initial-velocity))
- [ ]  12. Calibrate **UI crosshair and auto-aim box** ([§13](#13-operator-ui))
- [ ]  13. Check **referee system firmware and protocol** versions ([§15](#15-referee-system))
- [ ]  14. **Reset the C-board**

---

## 4. DJI (RM) Motors: 3508 / 2006

Applies to the chassis (3508 + C620), friction wheels (3508, no gearbox, + C620), and feeder (2006 + C610).

### 4.1 Set the motor ID

**What you need:** the DJI ESC manual (C620 or C610), the motor, the ESC, and power.

1. Look up the ID-setting procedure in the **DJI manual for that ESC** (C620 for 3508, C610 for 2006).
2. Set the ID according to the robot's motor assignment in code.
3. **Test the motor** before installing it on the robot.
4. Label the motor or ESC with its ID if possible.

### 4.2 Motor direction

- **Positive rotation = counterclockwise (CCW), viewed from above the output shaft.**
- If a motor needs to spin the other way, set this in its init config:

```c
.motor_reverse_flag = MOTOR_DIRECTION_REVERSE,
```

- Motors on opposite sides of a mechanism (for example, left and right friction wheels or radar mount motors) usually need **opposite** directions.

### 4.3 Zero point for DJI motors (yaw/pitch only)

DJI motors **cannot store a zero point**, so you read the encoder value with the debugger and write it into code. See [§6.2](#62-dji-rm-motors-for-yawpitch).

---

## 5. DM 4310 Motors (Yaw & Pitch)

> ⚠️ Disconnect the motor's **power and CAN from the robot** before any configuration (see [Safety](#1-safety-read-this-first)).
> 

### Before you start: open DM AC-Tools

Every step in this section uses DM AC-Tools.

1. Open **DM AC-Tools**. Download it from the RoboGrinder Google Drive: **2025-2026 → Embedded → software**.
2. Change the language using the **language toggle at the bottom** of the window (it shows **CN** when set to Chinese).

<img width="1704" height="1407" alt="image" src="https://github.com/user-attachments/assets/00d189de-ce85-4f7c-91ab-58fa0f64289a" />

### 5.1 Check the firmware version (do this first)

**Why:** If the motor isn't at its zero point after power-on and your code settings are correct, the firmware version is the likely cause. **Version V17 specifically fixes zero-point issues.**

1. Open DM AC-Tools and connect to the motor.
2. In the **Firmware Upgrade** panel, click **Read Version**. It shows the **Firmware REV** and **Bootloader REV**.
3. Compare the result with the latest firmware at [https://gitee.com/kit-miao/motor-firmware](https://gitee.com/kit-miao/motor-firmware).

**Reading DM firmware version numbers** *(translated from the DM firmware README)*:

DM motors use a **4-digit version number**. The **first two digits are the motor series code**, and the **last two digits are the version number**.

For example, in `APP_DM4310(V3)_V5017_04.bin`:

- `APP_DM4310(V3)` is the motor driver model the firmware is for.
- `50` means a **4310 motor with V3 hardware**.
- `17` is the **version number**.
- `_04` is the **sub-version**.

For the version numbers of other models, see **"固件更新日志-3.1.docx"** (*Firmware Update Log 3.1*).

`GM62_Sen_2005.bin`: the `GM62_Sen` prefix means this is **sensor firmware**. **Do not flash sensor firmware into the motor driver**, or the upgrade will fail.

> ⚠️ **Before upgrading, always read the current version first.** Choose firmware whose **first two digits match** the current version number. Otherwise, it can cause unknown problems (**even a failed upgrade and a motor that won't run**) and unnecessary trouble.
> 

**Example:** If the current version number is `6.3.1.3`, the motor is an **8006**, and its current version is **V13**. When upgrading, you can choose the **6317** firmware.

### 5.2 Upgrade the firmware (if needed)

Firmware Upgrade panel buttons:

| Button | What it does |
| --- | --- |
| Read Version (读版本) | Reads the current Firmware REV / Bootloader REV |
| Select Firmware (固件选择) | Choose the `.bin` file |
| Upgrade Firmware (固件升级) | Starts the upgrade |
| Firmware Name (固件名称) | Shows the selected file |
| Upgrade Progress (升级进度) | Progress bar |

1. Click **Read Version** and note the version.
2. Click **Select Firmware** and choose a `.bin` whose **first two digits match**.
3. Click **Upgrade Firmware** and wait until the progress reaches 100%.
4. Click **Read Version** again to confirm.
5. ⚠️ **After any firmware update, you must reset the motor's zero point** ([§5.4](#54-set-the-zero-point)).

### 5.3 Set the motor IDs

DM motors have **two** IDs. Set both:

- **Master ID (RX ID)**
- **Slave ID (TX ID)**
1. Connect the motor to DM AC-Tools.
2. Set the Master ID and Slave ID to match the robot's code.
3. Save the settings, then test the motor before installing it.

### 5.4 Set the zero point

The zero point is the encoder position that the code treats as 0°:

- **Pitch:** the barrel is **horizontal**
- **Yaw:** the turret and chassis both **face directly forward**

💡 Ask **ME** to print a **zero-point support bracket** so the position is exact.

1. Manually rotate the motor (or the mechanism) to the zero position, using the bracket.
2. In DM AC-Tools: **Debug → Enable → Save Zero Point**.
3. In code, set `middle_offset_float = 0.0f`, because the DM motor stores the zero point itself (see [§6.1](#61-dm-motors)).

### 5.5 Set PMAX / VMAX / TMAX

DM motors use these parameters to map our PID output values. They are set in **DM AC-Tools → Parameter Settings**, and they correspond to the **last three parameters of `DMMotorInit`** in code.

| Parameter | Meaning (from original doc) |
| --- | --- |
| PMAX | Maximum position range per revolution |
| VMAX | Maximum velocity |
| TMAX | Maximum torque |

> ⚠️ **For yaw, you MUST change PMAX.** Otherwise multi-revolution precision isn't handled correctly and the **yaw zero point drifts**.
> 
```c
yaw_motor = DMMotorInit(&yaw_config, DM_MOTOR_MODE_MIT_TORQUE, 3.141593f, 30.0f, 10.0f);
pitch_motor = DMMotorInit(&pitch_config, DM_MOTOR_MODE_MIT_TORQUE, 12.5f, 30.0f, 10.0f);
```
1. Open **Parameter Settings** in DM AC-Tools.
2. Set PMAX, VMAX, and TMAX.
3. Make sure the same values are in the last three arguments of `DMMotorInit` in code.

### 5.6 Find the angle limits

1. Enable the motor in DM AC-Tools.
2. Rotate the mechanism to its **maximum** angle and record the returned `position` value.
3. Rotate it to its **minimum** angle and record `position` again.
4. Enter both values into the limit macros (see [§6.3](#63-angle-limits)).

---

## 6. Gimbal Zero Point and Limits in Code

The gimbal motor init config has a calibration block like this:

```c
.cali_init_config = {
    .middle_offset_float = 0.0f,
    .max_limit = GIMBAL_PITCH_CALI_ECD_MAX_LIMIT,
    .min_limit = GIMBAL_PITCH_CALI_ECD_MIN_LIMIT,
```

### 6.1 DM motors

1. Set `middle_offset_float = 0.0f`. The zero point is already stored in the motor ([§5.4](#54-set-the-zero-point)).

### 6.2 DJI (RM) motors for yaw/pitch

> ⚠️ **Untested:** The original author never used DJI motors for yaw/pitch, so this procedure hasn't been verified and may need value conversions.
> 
1. Rotate the mechanism to the zero position.
2. Use the debugger to read the motor's **encoder value**.
3. Enter it into `middle_offset_float`.

### 6.3 Angle limits

Example values:

```c
#define GIMBAL_PITCH_CALI_ECD_MAX_LIMIT 0.33f
#define GIMBAL_PITCH_CALI_ECD_MIN_LIMIT -0.27f
```

- **DM motors:** Use the `position` values from [§5.6](#56-find-the-angle-limits).
- **DJI motors:** Use the same method as the zero point. Rotate to the max and min positions and record the encoder values (also untested).

---

## 7. C-Board Orientation and IMU Calibration

### 7.1 Set the C-board rotation matrix

The IMU readings must be rotated to match the robot's coordinate system (**ISO 8855**). The matrix depends on **how the C-board is physically mounted**.

The macro is in **`robot_def.h`**:

```c
// Rotation matrix that transforms the gyro & accelerometer into the ISO 8855 coordinate system
#define IMU_SENSOR_BOARD_INSTALL_ROTATION_MATRIX \
    {0.0f, 1.0f, 0.0f},  \
    {-1.0f, 0.0f, 0.0f}, \
    {0.0f, 0.0f, 1.0f}

#define IST8310_BOARD_INSTALL_SPIN_MATRIX \
    {1.0f, 0.0f, 0.0f}, \
    {0.0f, 1.0f, 0.0f}, \
    {0.0f, 0.0f, 1.0f}
```

You only change **`IMU_SENSOR_BOARD_INSTALL_ROTATION_MATRIX`** to match the mounting. Pick one of the two examples below.

**Mounting example A**, from the photo below:

<img width="960" height="773" alt="77d45676-d63f-4fe2-8082-b6ca95b16ab5" src="https://github.com/user-attachments/assets/71147cb8-2b4d-4feb-a5ff-94b416dd24f7" />

```c
    {0.0f, -1.0f, 0.0f}, \
    {1.0f,  0.0f, 0.0f}, \
    {0.0f,  0.0f, 1.0f}
```

**Mounting example B**, from the photo below:

<img width="932" height="591" alt="31deb630-6245-40ac-9eac-ce6cd26b34a2" src="https://github.com/user-attachments/assets/5482bd44-8245-40bd-ac3c-dd212ae55e1b" />

```c
    {0.0f,  1.0f, 0.0f}, \
    {-1.0f, 0.0f, 0.0f}, \
    {0.0f,  0.0f, 1.0f}
```

1. Check how the C-board is mounted on your robot.
2. Pick the matching matrix. If your mounting doesn't match either example, work it out using `BMI088.pdf` on the [Google Drive](https://drive.google.com/drive/folders/1FRRIVegkUYJ3QDxbZw7vjtt6OAHmZjjc).
3. Rebuild and flash the code.

> The diagrams are for reference only. See `2024-2025 → BMI088.pdf` for the full explanation.
> 

### 7.2 Calibrate the IMU

**How it works:** `BMI088_init(..., calibrate)` runs the calibration, saves the offsets to flash, and loads them on later boots. Here's the relevant part of `BMI088driver.c` (comments translated):

```c
if (calibrate)
{   // Calibration needed: keep the IMU LEVEL and STILL during calibration
    flashData.key = 0x12345678; // checks whether the flash sector is valid

    // Uncomment to calibrate and save into flash
    Calibrate_And_Save_MPU_Offset();          // ← (A) calibration line

    if (Load_Flash_Data())
    {
        // Load succeeded: read calibration values directly
        BMI088.TempWhenCali = flashData.TempWhenCali;   // ← (B) "success" line
        BMI088.AccelScale = flashData.AccelScale[0];
        BMI088.GyroOffset[0] = flashData.GyroOffset[0];
        ....
    }
    else
    {
        // Load failed: re-run blocking calibration
        Calibrate_MPU_Offset(&BMI088);        // ← (C) "failed" line
    }
}
```

**What you need:** the zero-point bracket (ask ME), a debug probe, and VS Code.

**Steps:**

1. In `BMI088driver.c`, **uncomment line (A)**: `Calibrate_And_Save_MPU_Offset();`
2. **Recompile** the code.
3. Install the **zero-point bracket**. Keep the robot **level and completely still** for the whole calibration.
4. Set these **4 breakpoints**. The limit is 5. The line numbers are approximate; put each breakpoint on the line with this code.
    
    
    | # | File | Code (≈ line) |
    | --- | --- | --- |
    | 1 | `robot.c` | `GimbalInit();` (≈ 32) |
    | 2 | `gimbal.c` | `INS_Init();` (≈ 42) |
    | 3 | `ins_task.c` | `while (BMI088_init(&hspi1, 1) != BMI088_NO_ERROR)` (≈ 143) |
    | 4 | `BMI088driver.c` | `Calibrate_And_Save_MPU_Offset();`, line (A) (≈ 152) |
    
    ```c
    // robot.c
    #if defined(ONE_BOARD) || defined(GIMBAL_BOARD)
        GimbalInit(); // ● breakpoint 1 (IMU init takes a while, so it goes first)
        ShootInit();
        RobotCMDInit();
    #endif
    ```
    
    ```c
    // gimbal.c
    void GimbalInit()
    {
        INS_Init(); // ● breakpoint 2 (IMU initializes first; its attitude data is used as the yaw motor's other feedback source)
        gimbal_IMU_data = INS_GetISO8855Attitude();
        ...
    ```
    
    ```c
    // ins_task.c
    while (BMI088_init(&hspi1, 1) != BMI088_NO_ERROR)  // ● breakpoint 3 (the "1" means calibrate)
        ;
    ```
    
    This is the call path the breakpoints follow:
    
    `GimbalInit()` → `INS_Init()` → `BMI088_init()` → `Calibrate_And_Save_MPU_Offset()`
    
5. Enter **debug mode**.
6. At every breakpoint, press **Step Into**, **except** at breakpoint 4.
7. At breakpoint 4 (`Calibrate_And_Save_MPU_Offset();`), press **Step Over**. This runs the whole calibration.
8. ✅ If the editor returns to `main.c` after the Step Over, **calibration is complete**.

### 7.3 Verify the calibration

1. **Comment line (A) back out** (`// Calibrate_And_Save_MPU_Offset();`).
2. **Recompile.**
3. **Remove** breakpoint 4. In `BMI088driver.c`, **add** breakpoints on:
    - line (B): `BMI088.TempWhenCali = flashData.TempWhenCali;` (≈ 157)
    - line (C): `Calibrate_MPU_Offset(&BMI088);` (≈ 170)
4. Enter debug mode and follow the same breakpoint procedure as above.
5. Press **Continue** until execution stops at **line (B)**, then press **Continue** again.
6. Wait a moment, then press **Stop**.

| Result | Meaning |
| --- | --- |
| ✅ Stops at **line (B)** | Calibration loaded from flash. **Success.** |
| ❌ Reaches **line (C)** | Flash load failed. **Calibration failed.** Repeat [§7.2](#72-calibrate-the-imu). |

### 7.4 IMU pitch limits

> This step may be unnecessary when the pitch motor is a DM motor. Once both the DM motor and the IMU are zeroed, the DM motor's limit values ([§5.6](#56-find-the-angle-limits)) can be used as the pitch limits.
> 
1. Flash the ELF with **Ozone**.
2. Watch the variable `imu_data.pitch`.
3. Raise the barrel to a reasonable **maximum** and record the value. Lower it to a reasonable **minimum** and record the value.
4. Enter both values into the IMU pitch upper and lower limit macros.

---

## 8. Chassis Parameters

Located in `robot_def.h`. **Get the measurements from the Mechanical branch!**

```c
// Robot chassis parameters
#define CHASSIS_WHEEL_RADIUS 0.0762f                        // wheel radius, in m
#define ONE_OVER_CHASSIS_WHEEL_RADIUS 13.1233595801f        // 1 / wheel radius
#define CHASSIS_WHEEL_DISTANCE_TO_CENTER 0.238f             // wheel-to-chassis-center distance, in m
#define ONE_OVER_CHASSIS_WHEEL_DISTANCE_TO_CENTER 4.20168067227f // 1 / that distance
```

1. Ask ME for the **wheel radius** and the **distance from each wheel to the chassis center**, both in meters.
2. Update both values.
3. **Recalculate the `ONE_OVER_...` values** as 1 ÷ the value. Example: 1 / 0.0762 = 13.1233595801.

> 💡 We store `1/x` ahead of time because division is slow on microcontrollers and multiplication is fast.
> 

---

## 9. Pitch Gravity Compensation

Gravity compensation is a **feedforward torque** that cancels gravity on the pitch assembly, so the PID doesn't have to fight it.

### 9.1 Get the numbers from SolidWorks

1. Open the pitch assembly in SolidWorks (ask ME).
2. **Evaluate tab → Mass Properties** (or **Tools → Evaluate → Mass Properties**). Record the **mass** of the pitch section, **excluding the motor**.
    
    <img width="1950" height="1489" alt="3" src="https://github.com/user-attachments/assets/db005317-8807-4600-afc7-c351c58423d8" />
    
3. Use **Measure** to find **dz**, the distance from the **center of mass** to the **center of the motor's rotation axis**.
    
    <img width="1588" height="950" alt="4" src="https://github.com/user-attachments/assets/1b2ff50d-7a45-4b0a-b1fd-1a81d5f5b70b" />

### 9.2 Calculate the torque

```
torque (N·m) = mass (kg) × 9.81 × dz (m)
```

**Worked example (INF26 pitch assembly):**

- Mass = 1707.95 g = **1.70795 kg**
- dz = 73.05 mm = **0.07305 m**
- Torque = 1.70795 × 9.81 × 0.07305 ≈ **1.224 N·m**

This is the value in `gimbal.c` (the sign depends on the motor direction):

```c
// Gimbal feedforward control
static void gimbal_feedfoward_control()
{
    // If the pitch motor is offline, skip the gravity feedforward
    if (DaemonIsError(pitch_motor->daemon))
    {
        pitch_torque_feedforward = 0.0f;
    }
    else
    {
        // Method 1: IMU frame, cos(Roll) accounts for roll angle
        // pitch_torque_feedforward = -0.5 * 14634.0f * GIMBAL_PITCH_MASS * 9.81f * GIMBAL_PITCH_COM_DISTANCE * arm_cos_f32(gimbal_IMU_data->Pitch) ...

        // Method 2: motor frame
        pitch_torque_feedforward = -1.2235f;
        // pitch_torque_feedforward = -0.5 * 14634.0f * GIMBAL_PITCH_MASS * 9.81f * GIMBAL_PITCH_COM_DISTANCE * arm_cos_f32(pitch_motor->measure.angle_single...) ...
    }
}
```

> The calculated value is a **starting point**. Friction, cable tension, and cable weight aren't in the CAD model, so you'll likely need to adjust it.
> 

`robot_def.h` also has pitch mass and center-of-mass macros:

```c
#define GIMBAL_PITCH_MASS 1.34f         // mass above the pitch axis, for gravity feedforward, in kg
#define GIMBAL_PITCH_COM_DISTANCE 0.06f // distance from that center of mass to the pitch motor axis, in m
```

> ⚠️ These are **only used by the commented-out formulas** (Method 1 and the commented line in Method 2). The active code uses the hardcoded `-1.2235f`. They also **don't match** the INF26 SolidWorks numbers (1.708 kg, 0.073 m). If you switch to a formula-based method, update these macros first.
> 

### 9.3 Test it

1. Use the **INF version** of `robot_cmd.c`.
2. Set **all pitch MANUAL PID parameters to `0.0f`**. This disables the PID and leaves only the feedforward.
3. ✅ **Pass if:** the pitch **holds still at any angle**, or feels **extremely light** when you move it up and down by hand.
4. If it falls, increase the torque. If it rises, decrease it.

**Known limitations (TODO):**

- [ ]  Assumes the robot is **perfectly level** (no roll). If the robot tilts, the needed torque changes.
- [ ]  Ignores how the lever arm changes as the barrel tilts up and down, so the balance point is only accurate near **horizontal**.

---

## 10. PID Tuning (Pitch & Yaw)

> ⚠️ **Connect the wireless kill switch before tuning ANY PID.** Motors can suddenly run away.
> 

**Tools:** **J-Link + Ozone** for live variables. You'll also need the RM operator client (for MANUAL tuning) and RQT Plot (for AUTO tuning).

### 10.1 Where the parameters are

Located in `pitch.h` (yaw is set up the same way):

<img width="507" height="591" alt="5" src="https://github.com/user-attachments/assets/9bf3797a-bf78-41d3-a65e-a4ba65d20434" />

Naming pattern: `GIMBAL_{YAW|PITCH}_{ANGLE|SPEED}_PID_{KP|KI|KD}_{MANUAL|AUTO}`

Example values (current Infantry):

| Loop | Kp | Ki | Kd |
| --- | --- | --- | --- |
| Yaw angle (manual) | 8.5 | 0.0 | 0.0 |
| Yaw speed (manual) | 1.5 | 0.000025 | 0.0 |
| Yaw angle (auto) | 7.0 | 0.0 | 0.0 |
| Yaw speed (auto) | 0.9 | 0.0 | 0.0 |
| Pitch angle (manual) | 9.0 | 1.5 | 0.0 |
| Pitch speed (manual) | 1.65 | 0.0263 | 0.0 |
| Pitch angle (auto) | 2.0 | 0.0 | 0.0 |
| Pitch speed (auto) | 0.9 | 0.0 | 0.0 |

- **MANUAL** = the operator is driving
- **AUTO** = auto-aim is controlling the gimbal

### 10.2 Tuning order

1. **Speed loop first.** Change the motor's **outer loop from the angle loop to the speed loop**, with a **target speed of `0.0f`** .
2. Tune **Kp**, then **Kd**, then (optionally) **Ki** using the method below.
3. **Then switch back to the angle loop** and tune its PID using the same method.

#### 10.2.1 Switching to speed loop for tuning

In the gimbal control code (`gimbal.c`), the **normal** setup is:

```c
DMMotorEnable(yaw_motor);
DMMotorEnable(pitch_motor);
// DMMotorOuterLoop(yaw_motor, SPEED_LOOP); // for pid tuning
DMMotorChangeFeed(yaw_motor, SINGLE_ROUND_ANGLE_LOOP, OTHER_FEED); // comment this line when tuning speed_loop pid
DMMotorChangeFeed(yaw_motor, SPEED_LOOP, OTHER_FEED);
// DMMotorOuterLoop(pitch_motor, SPEED_LOOP);
DMMotorChangeFeed(pitch_motor, SINGLE_ROUND_ANGLE_LOOP, OTHER_FEED);
DMMotorChangeFeed(pitch_motor, SPEED_LOOP, OTHER_FEED);
DMMotorSetRef_mode_mit_torque(yaw_motor, gimbal_cmd_recv.yaw);     // multi-turn / single-turn handled in the motor module
// DMMotorSetRef_mode_mit_torque(yaw_motor, 0.0f);
DMMotorSetRef_mode_mit_torque(pitch_motor, gimbal_cmd_recv.pitch);
// DMMotorSetRef_mode_mit_torque(pitch_motor, 0.0f);
break;
```

**To tune the speed loop** for a motor (yaw shown; pitch is the same with `pitch_motor`):

1. **Uncomment** `DMMotorOuterLoop(yaw_motor, SPEED_LOOP);`. This sets the outer loop to the speed loop.
2. **Comment out** `DMMotorChangeFeed(yaw_motor, SINGLE_ROUND_ANGLE_LOOP, OTHER_FEED);`.
3. **Set the target to 0:** comment out `DMMotorSetRef_mode_mit_torque(yaw_motor, gimbal_cmd_recv.yaw);` and uncomment `DMMotorSetRef_mode_mit_torque(yaw_motor, 0.0f);`.
4. Recompile, flash, and tune.
5. ⚠️ **When you're done, undo all three changes** to go back to the angle loop.

> 💡 Tune **one motor at a time**. Only change the lines for the motor you're tuning.
> 

> 💡 Whether the innermost loop is a current loop or another type depends on what input the motor expects.
> 

### 10.3 Tuning Kp (step by step)

**Goal:** find the **critical point**, where Kp goes from *no jitter* to *jitter*, then back off a little.

1. **Starting value:** Take the previous Kp for the same motor model and round it.
    - Example: 12.123f becomes **12.0f**. 0.123f becomes **0.1f**.
2. Set **Ki = 0.0f** and **Kd = 0.0f**.
3. **Coarse search (Step 1):** If there's **no oscillation, increase** Kp. If there **is** oscillation, **decrease** it.
    - If **Kp > 1.0**, change it by **±2.0**. Example: 10.0f → 12.0f.
    - If **Kp < 1.0**, change it by **±0.2**. Example: 0.5f → 0.3f.
    - Keep going until you find a range where the lower end has **no jitter** and the upper end **has jitter**.
4. **Fine search (Step 2): binary search** inside that range, down to 3–4 decimal places.
    - Example: The range is 10.0–12.0. Try **11.0**.
        - If 11.0 jitters, try **10.5**.
        - If 10.5 doesn't jitter, try **10.75**. Keep halving the range.
5. **Final Kp = critical point × 0.9** (or slightly lower). Don't leave Kp sitting right on the edge.

### 10.4 Tuning Kd

1. Starting guess: **Kd ≈ 0.1 × Kp** (not a fixed rule).
2. Use the **same coarse + binary search** to find the Kd threshold, from *no jitter* to *slight high-frequency jitter*.
    - ⚠️ Too much Kd usually shows up as **slight, high-frequency jitter**. If you see that, Kd is too high. Lower it to find the threshold.
3. **Final Kd = threshold × 0.9** (or slightly lower).

### 10.5 Tuning Ki (optional)

- Ki usually isn't needed. If you use it, start around **Ki ≈ 0.05 × Kp**.

### 10.6 MANUAL vs. AUTO tuning

| Mode | How to judge it |
| --- | --- |
| **MANUAL PID** | Ask the **operator** using the RM operator client: Too fast? Too slow? Unresponsive? Overshoot? Jitter? 💡 Control sensitivity can be changed in the settings panel (press **P**). |
| **AUTO PID** | Work with the **auto-aim team** using **RQT Plot**. Watch the waveforms for **tracking delay and error** (some steady error is expected when Ki = 0). You can use the test script by Britny and Mikiya. |

**Yaw** is tuned the same way.

---

## 11. Feeder (Bullet Feed) Debugging

The feeder (loader / magazine wheel) code is in **`shoot.c`**. The feeder logic settings are `#define`s near the **top of `shoot.c`**, and the motor setup is in `ShootInit()`.

`robot_def.h` also has shooter parameters:

```c
// Shooter parameters
#define SHOOT_FRICTION_WHEEL_RADIUS 0.03f                   // friction wheel radius, in m
#define ONE_OVER_SHOOT_FRICTION_WHEEL_RADIUS 33.3333333333f // 1 / friction wheel radius

#define SHOOT_ONE_BULLET_ANGLE 0.698131701f // angle per bullet, in rad

#define SHOOT_FEEDING_FAN_NUM_OF_BULLETS_PER_ROUND 9.0f              // bullets per loader revolution
#define ONE_OVER_SHOOT_FEEDING_FAN_NUM_OF_BULLETS_PER_ROUND 0.11111f // 1 / bullets per revolution
```

> ⚠️ **The angle per bullet is defined twice:** `SHOOT_ONE_BULLET_ANGLE` in `robot_def.h` and `LOADER_ONE_BULLET_ANGLE` in `shoot.c`. The current `shoot.c` uses **its own** `LOADER_ONE_BULLET_ANGLE`. If you change the loader, **update both** (and the bullets-per-round values) so they stay consistent.
> 

### 11.1 Hardware setup (in `ShootInit()`)

| Setting | Current value | Notes |
| --- | --- | --- |
| Motor | `M2006` (+ C610) |  |
| CAN bus | `hcan2` |  |
| CAN ID | `.tx_id = 8` | Must match the ESC's ID ([§4.1](#41-set-the-motor-id)) |
| Direction | `MOTOR_DIRECTION_NORMAL` | Set it so the positive direction **pushes a bullet out** |
| Holes in the loader | 9 | Set by `LOADER_ONE_BULLET_ANGLE` (below) |

### 11.2 How the feeder works (read this before tuning)

The loader moves in **phases**. With a 9-hole loader, one phase = one bullet = **40°**:

```c
#define LOADER_ONE_BULLET_ANGLE 0.6981317f  // 2*pi/9, 9-hole loader, 40° per bullet
#define LOADER_DONE_THRESHOLD   0.05f       // "arrived" threshold (rad)
#define LOADER_JAM_ANGLE_ERR    0.10f       // single-shot jam check: only call it a jam if still this far from target (rad)
#define LOADER_PHASE_OFFSET     0.0f        // offset from the mechanical zero to the ideal resting phase; tune on the real robot
#define LOADER_AUTO_DEAD_TIME_MS -1.0f      // delay between shots in auto-aim fire mode
```

> If your loader has a different number of holes, change `LOADER_ONE_BULLET_ANGLE` (in `shoot.c`) to **2π ÷ (number of holes)**. Also update `SHOOT_ONE_BULLET_ANGLE`, `SHOOT_FEEDING_FAN_NUM_OF_BULLETS_PER_ROUND`, and `ONE_OVER_SHOOT_FEEDING_FAN_NUM_OF_BULLETS_PER_ROUND` in `robot_def.h`.
> 

**Key ideas:**

- **The M2006 has no absolute zero.** The code treats the loader position at its **first run after power-on** as the zero point. So, **before powering on, manually turn the loader paddle to its ideal starting position.** If it starts out of position, every shot after that is off too (empty shots, double shots, or jams).
- **Every target is calculated from the zero**: target = zero + phase number × 40°. It's *not* "current angle + 40°", so small errors don't add up over time.
- **When stopped, the loader holds its angle** (angle loop), so bullets or spring-back can't slowly push it out of position.
- **When you let go of continuous fire**, the loader moves forward to the **next legal phase** and holds there.
- **Turning the friction wheels off and on doesn't reset the hold**, so the loader doesn't slowly creep after re-enabling.

> 💡 This is a software-only zero. The long-term plan is to add a **Hall or optical zero sensor** so the loader zeroes itself automatically.
> 

**Fire modes** (`load_mode` sent from `robot_cmd.c`):

| Mode | What it does |
| --- | --- |
| `LOAD_STOP` | Holds the current phase (angle loop). Before the first shot, it uses speed loop at 0 instead, to avoid jitter. |
| `LOAD_1_BULLET` | Moves forward **one phase**, only on a **new** button press (rising edge), so holding the button doesn't fire repeatedly. 500 ms between shots. Used for the energy mechanism and the Hero. |
| `LOAD_AUTO` | Auto-aim fire. Moves one phase, with delay `LOADER_AUTO_DEAD_TIME_MS`. |
| `LOAD_BURSTFIRE` | Continuous fire. Steps forward one phase at a time at the commanded fire rate (`shoot_rate`, bullets/s). |
| `LOAD_REVERSE_CALI` | Reverse for calibration (different from jam reversing). |

### 11.3 Continuous fire settings

```c
#define LOADER_BURST_MAX_LEAD_PHASES 2.0f   // target can be at most 2 phases ahead of the actual position
#define LOADER_BURST_MIN_INTERVAL_MS 20U    // minimum time between shots (max 50 shots/s)
```

- If the loader falls behind (for example, it's struggling against a bullet), it won't keep adding targets. The target stays at most **2 phases ahead**.

### 11.4 Heat protection

The code stops firing **before** the referee heat limit is reached:

```c
#define SHOOT_HEAT_PER_BULLET        10U   // heat added per bullet
#define SHOOT_HEAT_RELEASE_MARGIN    35U   // once locked, firing resumes only when there's this much room left
#define SHOOT_LOW_HEAT_THRESHOLD     50U   // when remaining heat is at or below this...
#define SHOOT_LOW_HEAT_RATE          8.0f  // ...continuous fire is slowed to this rate (bullets/s)
```

- **Lock:** If the next bullet would reach the heat limit, firing stops and the loader holds its position.
- **Unlock:** Firing resumes only when there's room for the next bullet **plus** `SHOOT_HEAT_RELEASE_MARGIN`. This prevents rapid lock/unlock flickering.
- **Slow-down:** When remaining heat drops to `SHOOT_LOW_HEAT_THRESHOLD` or less, continuous fire is capped at `SHOOT_LOW_HEAT_RATE`.
- Heat protection is **skipped** when the referee data isn't usable: `ref_status != 0` (in `robot_def.h`, **0 = normal, 1 = unavailable**) or the heat limit reads 0.

The heat values come from `robot_cmd.c` through `Shoot_Ctrl_Cmd_s` (in `robot_def.h`):

```c
uint8_t  ref_status;      // 0: normal, 1: unavailable
uint16_t ref_heat_limit;
uint16_t ref_heat;
uint16_t ref_heat_offset; // reserved heat (offset)
```

> 💡 **Testing continuous fire on the auxiliary robot:** If firing keeps cutting out, check whether heat protection is what's stopping it.
> 

### 11.5 Jam detection and recovery

There are **two** jam-recovery paths.

**1. Normal jam (all modes):** set in `ShootInit()`:

```c
set_shooter_info(0.88f, 400, 200, 8.37f);
//               │      │    │    └─ reverse speed (rps)
//               │      │    └────── reverse time (ms)
//               │      └─────────── how long it must be stuck before reversing (ms)
//               └────────────────── "stuck" speed threshold (rps)
```

- For single shots, it only counts as a jam if the loader is **slow AND still more than `LOADER_JAM_ANGLE_ERR` from its target**. (Being slow near the target is normal.)

**2. Fast reverse (continuous fire only):** for "hard jams," where a bullet gets stuck between the loader and the paddle. Waiting 400 ms for the normal path leaves a long gap in fire, so this reacts faster:

```c
#define LOADER_BURST_HARD_JAM_SPEED_RPS  0.75f  // below this speed counts as "almost stalled"
#define LOADER_BURST_HARD_JAM_TIME_MS    100U   // must stay almost stalled this long to trigger
#define LOADER_BURST_MIN_CMD_SPEED_RPS   1.00f  // only checks when the commanded speed is at least this
#define LOADER_FAST_REVERSE_TIME_MS      100U   // how long to reverse
#define LOADER_FAST_REVERSE_IGNORE_MS    80U    // after reversing, ignore jam checks this long
```

**How to tune the fast reverse:**

| Problem | Change |
| --- | --- |
| Triggers when there's no jam (false triggers) | **Lower** `HARD_JAM_SPEED_RPS` and/or **raise** `HARD_JAM_TIME_MS` |
| Doesn't catch real jams fast enough | **Raise** `HARD_JAM_SPEED_RPS` and/or **lower** `HARD_JAM_TIME_MS` |
| Reversing messes up bullet positions | Keep `FAST_REVERSE_TIME_MS` short |

**Debug counters:** Watch these in **Ozone** to see which path is handling jams:

| Variable | Counts |
| --- | --- |
| `debug_loader_fast_reverse_count` | Fast reverses (continuous fire) |
| `debug_loader_normal_jam_count` | Normal 400 ms jam reverses |

> 💡 If there's still a ~0.6 s gap in fire during jams but `debug_loader_fast_reverse_count` barely goes up, the fast reverse isn't catching them. Make it more sensitive.
> 

### 11.6 Feeder PID

In `ShootInit()` → `loader_config`. Current values:

| Loop | Kp | Ki | Kd | MaxOut |
| --- | --- | --- | --- | --- |
| Angle | 52.0 | 0.0 | 4.0 | 30.0 |
| Speed | 1700.0 | 0.02 | 0.0 | 10000.0 |

- The angle loop's `DeadBand` is set to `LOADER_DONE_THRESHOLD`, so they always match.
- **Angle Ki is 0 on purpose** during debugging. Integral can make the loader jitter or creep while holding with no load. Once the phase logic is stable, you can try **0.1–0.3**.

### 11.7 Safety stops

The loader **stops firing** automatically if:

- **Both** friction wheels are offline, or
- The loader motor is offline, or
- The friction wheels are turned off.

> The friction-wheel **low-speed** protection exists in the code but is currently **commented out**.
> 

### 11.8 Steps

1. Check the **CAN ID and direction** ([§11.1](#111-hardware-setup-in-shootinit)).
2. **Power off.** Manually turn the loader paddle to its ideal starting position.
3. Power on and enable the shooter.
4. **Single shot test:** Fire several single shots. ✅ Each shot moves exactly one phase (40°) and fires exactly **one** bullet. If the resting position is consistently off, adjust `LOADER_PHASE_OFFSET`.
5. **Continuous fire test:** Hold fire, then release. ✅ The fire rate matches `shoot_rate`, and the loader stops at a clean phase.
6. **Jam test:** Run continuous fire for a while and watch the debug counters in Ozone. Tune the fast reverse if needed ([§11.5](#115-jam-detection-and-recovery)).
7. If the loader jitters or creeps while holding, check the angle PID ([§11.6](#116-feeder-pid)).

---

## 12. Projectile Initial Velocity

### 12.1 Where the value is

The friction wheel speed is set in **`shoot.c` → `ShootTask()`**, as a single **linear speed in m/s**:

```c
// Turn on the friction wheels first
if (shoot_cmd_recv.friction_mode == FRICTION_ON)
{
    // TODO: switch to the linear-fit output / radius later
    // float required_wheel_linear_speed = (DESIRED_BULLET_SPEED - WHEEL_TO_BULLET_B) / WHEEL_TO_BULLET_A;
    float required_wheel_linear_speed = 17.0f; // m/s   ← change this

    FrictionWheelSetSyncRef(required_wheel_linear_speed);
}
```

> 📌 **This changed from older versions.** The code used to pick a speed with a `switch` on `bullet_speed_type`. The current `shoot.c` doesn't use that anymore. Each robot's `shoot.c` sets its own speed with `required_wheel_linear_speed`.
> 
> 
> The speed enum still exists in `robot_def.h`, and `bullet_speed_type` is still sent from `robot_cmd.c` in `Shoot_Ctrl_Cmd_s`, but **`shoot.c` ignores it**. Changing it does nothing to the actual speed.
> 
> ```c
> typedef enum
> {
>     BULLET_SPEED_NONE = 0,
>     BIG_AMU_10,
>     BIG_AMU_12,
>     SMALL_AMU_15,
>     SMALL_AMU_18,
>     SMALL_AMU_30,
> } shoot_bullet_speed_type_e;
> ```
> 

The friction wheel radius is in `robot_def.h`: `SHOOT_FRICTION_WHEEL_RADIUS 0.03f` and `ONE_OVER_SHOOT_FRICTION_WHEEL_RADIUS 33.3333333333f`. If ME changes the wheel size, update **both**.

### 12.2 Steps

1. Open `shoot.c` for your robot.
2. Change `required_wheel_linear_speed` (m/s).
3. Flash the code and **measure the actual projectile speed** (e.g., with the referee system's speed readout).
4. Adjust and repeat until the speed is **safely below the rule limit**. Example: with a 25 m/s limit, keep it **under 23 m/s**.
5. If the actual bullet speed changed noticeably, update the **auto-aim predicted speed** too ([§12.5](#125-auto-aim-predicted-bullet-speed)).

> ⚠️ **Speed increases as the friction wheels heat up** during long continuous fire. Leave margin so the speed doesn't go over the limit.
> 

### 12.3 Left/right friction wheel sync

If one friction wheel spins faster than the other, the bullet curves. `FrictionWheelSetSyncRef()` corrects for this: it compares the two wheels' actual linear speeds and nudges them toward each other.

```c
#define FRIC_SYNC_KP          0.3f   // correction strength
#define FRIC_SYNC_KD          0.0f
#define FRIC_SYNC_COMP_LIMIT  0.3f   // max correction (m/s), so one bad wheel can't drag the other too far
#define FRIC_SYNC_LPF_ALPHA   0.15f  // low-pass filter on the speed difference (smaller = smoother but slower)
#define FRIC_SYNC_DEADBAND    0.15f  // ignore differences smaller than this (m/s), to avoid back-and-forth jitter
```

- If the **left** wheel is faster, it lowers the left target and raises the right target (and the reverse).
- The function converts the final **linear** speeds (m/s) to **angular** speeds before sending them to the motors, so you only ever work in m/s.

**Check it:** In **Ozone**, watch `debug_fric_speed_diff`, the left/right speed difference.

> ⚠️ `debug_fric_speed_diff` is declared as an `int`, so it only shows **whole** m/s. A difference of 0.5 m/s shows as **0**. To see small differences, watch `shoot_feedback_data.friction_l_v` and `shoot_feedback_data.friction_r_v` instead.
> 

### 12.4 Friction wheel motor setup

In `ShootInit()` → `friction_config`:

| Setting | Left | Right |
| --- | --- | --- |
| Motor | `M3508_NO_GEAR_BOX` | `M3508_NO_GEAR_BOX` |
| CAN bus | `hcan2` | `hcan2` |
| CAN ID | `tx_id = 1` | `tx_id = 2` |
| Direction | `MOTOR_DIRECTION_REVERSE` | `MOTOR_DIRECTION_NORMAL` |

Speed PID (both wheels): **Kp 230.0, Ki 1.0, Kd 50.0, MaxOut 11000.0**. Friction wheel diameter is 60 mm (radius 0.03 m).

### 12.5 Auto-aim predicted bullet speed

```c
SetEmbed2PC_shoot_bullet_speed(20.0f); // predicted speed TODO: could be made dynamic
```

This tells the **MiniPC (auto-aim)** what bullet speed to use for trajectory prediction. It's **fixed at 20 m/s** and doesn't update automatically when you change `required_wheel_linear_speed`. If the measured bullet speed is noticeably different, **update this value and tell the software team.**

### 12.6 Linear fit (not in use yet)

There's a planned way to set the wheel speed from a **desired bullet speed**, using a measured linear fit:

```c
#define WHEEL_TO_BULLET_A 0.0617f      // linear fit slope
#define WHEEL_TO_BULLET_B 9.83f        // linear fit intercept
#define DESIRED_BULLET_SPEED 19.0f     // desired bullet speed (m/s)
```

It's **commented out**. ⚠️ Check the units before turning it on: (19.0 − 9.83) ÷ 0.0617 ≈ **148.6**, which isn't a realistic wheel speed in m/s. The fit was probably measured in different units (such as rad/s or RPM).

> 💡 **What "1/radius" means:** angular velocity = linear velocity ÷ radius = linear velocity × (1 / radius). `ONE_OVER_SHOOT_FRICTION_WHEEL_RADIUS` is precomputed because division is slow on microcontrollers.
> 

---

## 13. Operator UI

### 13.1 Drawing function

```c
/* Draw a straight line
 * graph         pointer to a Graph_Data, stores the graphic data
 * graphname[3]  graphic name (3 chars), identifies it for later changes
 * Graph_Operate graphic operation (see header file), e.g. UI_Graph_ADD
 * Graph_Layer   layer 0–9
 * Graph_Color   color
 * Graph_Width   line width
 * Start_x, Start_y   start point
 * End_x, End_y       end point
 */
void UILineDraw(Graph_Data_t *graph, char graphname[3], uint32_t Graph_Operate,
                uint32_t Graph_Layer, uint32_t Graph_Color, uint32_t Graph_Width,
                uint32_t Start_x, uint32_t Start_y, uint32_t End_x, uint32_t End_y);
```

**UI coordinate system:**

```
y
^
|
|
0 - - - - - > x
```

> The origin (0, 0) is the bottom-left of a 1920×1080 screen, so the crosshair center is x = 960, y = 540. A smaller y is lower on screen, which is why the auto-aim box uses - 30 to move down.
> 

### 13.2 Crosshair

Located in `application/referee_interaction/referee_ui_task.c` → `static_UI_func()`:

```c
// Aim crosshair (static lines)
// Center vertical line
UILineDraw(&gAimVertL,    "001", UI_Graph_ADD, 2, UI_Color_Orange, 1, 960, 540 - 8, 960, 500 - 8);
// 2 m (small armor plate)
UILineDraw(&gAimHorizL2m, "003", UI_Graph_ADD, 2, UI_Color_Yellow, 1, (960 - 30 - 9), (540 - 24), (960 + 30 - 9), (540 - 24));
// 4 m
UILineDraw(&gAimHorizL4m, "004", UI_Graph_ADD, 2, UI_Color_Yellow, 1, (960 - 15 - 9), (540 - 45), (960 + 15 - 9), (540 - 45));
// 5 m
UILineDraw(&gAimHorizL5m, "005", UI_Graph_ADD, 2, UI_Color_Yellow, 1, (960 - 12 - 9), (540 - 66), (960 + 12 - 9), (540 - 66));
// 7 m
UILineDraw(&gAimHorizL7m, "006", UI_Graph_ADD, 2, UI_Color_Yellow, 1, (960 - 9 - 9),  (540 - 95), (960 + 9 - 9),  (540 - 95));
// 8 m
UILineDraw(&gAimHorizL8m, "007", UI_Graph_ADD, 2, UI_Color_Green,  4, (960 - 7), (540 - 8), (960 + 7), (540 - 8));
```

- Each graphic needs a **unique 3-character name** (`"001"`, `"003"`, and so on).
- The commented-out lines in the source are the **big armor plate** versions.

**Drawing a line isn't enough. You also have to send (refresh) it.** Further down in the same file:

```c
// Infantry only uses the crosshair center
UIGraphRefresh(get_referee_info_referee_id_ptr(), 2, gAimVertL, gAimHorizL8m);
// UIGraphRefresh(get_referee_info_referee_id_ptr(), 5, gAimVertL, gAimHorizL2m, gAimHorizL4m, gAimHorizL5m, gAimHorizL7m);
// UIGraphRefresh(get_referee_info_referee_id_ptr(), 5, gAimHorizL8m, left8to7, left7to5, left5to4, left4to2);
// UIGraphRefresh(get_referee_info_referee_id_ptr(), 2, right8to7, right7to5);
// UIGraphRefresh(get_referee_info_referee_id_ptr(), 2, right5to4, right4to2);
```

- The 2nd argument is the **number of graphics** being sent, and it must match how many you list after it.
- Currently the Infantry only sends the **vertical line + the 8 m center mark**. To show the other distance lines, uncomment the matching `UIGraphRefresh` calls.
- If a line you drew doesn't appear on screen, check that it's included in a `UIGraphRefresh` call.

<img width="1063" height="260" alt="11" src="https://github.com/user-attachments/assets/547d461a-e22a-471a-a4b3-71a3a234a749" />

**Calibration steps:**

1. Fire at a target **3 m** away and note the impact point on the operator screen.
2. Move the line coordinates so the **horizontal line crosses the vertical line at the 3 m impact point**.
3. Extend the vertical line far enough to cover impact points at **5 m and beyond**.
4. Repeat for each distance line.
5. ✅ **Done when:** the lines match the impact points **and the operator is satisfied**.

> ⚠️ **Tell ME not to remove or reinstall the video transmission camera after the crosshair is calibrated.** Any reinstall introduces error, so the crosshair must be **recalibrated**. Communicate this clearly with ME.
> 

### 13.3 Auto-aim FOV box

Located in `referee_ui_coordinate_info.h`:

```c
// Auto aim box coordinate
#define Auto_Aim_Box_Start_X (Center_X - 200)
#define Auto_Aim_Box_End_X   (Center_X + 200)
#define Auto_Aim_Box_Start_Y (Center_Y + 200 - 30)
#define Auto_Aim_Box_End_Y   (Center_Y - 200 - 30)
```

- The box is 400 × 400 around the screen center. The `30` shifts it **down**, because the auto-aim camera sits below the video transmission camera.
- To adjust the box, change the `200` (half-size) and `30` (vertical offset) values.
- In the Season 26 design, the auto-aim camera sits **below** the video transmission camera, so the box is **slightly below center** on screen.
- ✅ **Standard:** The `Auto_Aim_Box` roughly matches the auto-aim camera's FOV at **3 m**. Armor plates outside the box can't be recognized.

### 13.4 UI tips

- Pay attention to UI-specific changes in the **communication protocol** each year.
- **Bandwidth:** Too much UI data makes the UI lag.
- **C-board load:** Too many elements or updates sent too often will noticeably raise CPU usage.
- **UI send settings** are in `robot_def.h`:
    
    ```c
    #define REFEREE_SEND_USE_DMA 0          // send UI data with DMA: 1 = yes, 0 = no
    #define REFEREE_SEND_MODE_DMA_DELAY 115 // DMA send delay, in ms
    #define UI_ADD_FRAME_BREAK_TIME 30      // gap between UI "add" frames, in ms
    #define UI_CHANGE_FRAME_BREAK_TIME 30   // gap between UI "change" frames, in ms
    ```
    
    If the UI lags or elements don't show up, these frame gaps are one place to look.
    
- Add or remove other UI elements as needed.

---

## 14. OLED Screen

### 14.1 Display orientation

- Change the display orientation in `oled.c` to match how the screen is mounted. If you're not sure how, look it up with ChatGPT.

### 14.2 What the OLED shows

**Alive check:** If the numbers in the **top-left and top-right corners keep refreshing**, the C-board is running. The top-left number is battery level, but it's **not accurate**.

| Label | Meaning |
| --- | --- |
| FL | Friction wheel (flywheel), left |
| FR | Friction wheel (flywheel), right |
| M1–M4 | Chassis motors |
| yaw | Yaw motor |
| pL | Pitch motor, left (main) |
| pR | Pitch motor, right (not present on robots with a single DM 4310 pitch) |
| tL | None (not used; Season 26 had no dual-barrel robots) |
| tR | Right feeder motor |
| GYR | Gyroscope |
| ACC | Accelerometer |
| MAG | Magnetometer |
| REF | Referee system |
| SP | Supercapacitor (see below) |
| DBUS | Remote controller receiver |

**Reading the SP (supercapacitor) status:**

| What you see | Meaning |
| --- | --- |
| Check mark gone, but a number is still shown | **Abnormal** |
| Number is 0 and no check mark | **Disconnected (offline)** |

- Adjust what's displayed each season as needed.
- 💡 **First debugging step when a working robot malfunctions:** check the OLED to see whether the hardware for that function is **online**.

---

## 15. Referee System

### 15.1 Referee firmware (RMTool2)

**Download:** [https://bbs.robomaster.com/wiki/20204847/811111?source=7](https://bbs.robomaster.com/wiki/20204847/811111?source=7)

Rules:

1. All referee system modules must share the **same major version**. Example: don't mix v11.x.x.x with v6.x.x.x.
2. The **video transmission** and **operator terminal** major versions must match, or there may be **no video feed**.
3. The **latest firmware isn't always best.** Example: main controller firmware **12.0.0.7** makes buffer energy go **negative**, which **locks the chassis**. Use **12.0.0.3** instead.

**Steps to flash:**

1. Open RMTool2 and connect the referee module.
2. In the firmware list, **double-click** the version you want. The version field turns **green** when it's selected.
3. Start the flash.
4. If the computer says “Update Successful” but the controller stays stuck on “Updating…”, **wait until it's clearly frozen**, then disconnect and power the robot back on.
5. After powering on, **check that the version number is correct**.

### 15.2 Yearly referee system and terminal upgrades

- Check for referee system and operator terminal upgrades **every spring**.
- Resources: [https://bbs.robomaster.com/wiki/20204847](https://bbs.robomaster.com/wiki/20204847). This link may change; otherwise go to **RoboMaster website → College Series → College League → Rules/Resource Center**, or the **ARCC** website.
- About **one week before each competition**, the referee system and protocol versions for that event are announced on Discord or the official website (this year it was in the **Site Manual** section of the rules). **Check on time** and make sure our code matches.

### 15.3 Updating the communication protocol

The protocol **changes every year**. Update our code, then **tell the software team** what changed so they can update their side.

**Files:**

| File | What it does | Change it? |
| --- | --- | --- |
| `rm_referee.c` | Serial settings and unpacking of referee messages | Usually **no** |
| `rm_referee.h` | `referee_cmd_id_t` enum | **Yes**, add new command IDs |
| `referee.c` | Maps referee data to structs, plus `get` functions | **Yes** |
| `referee.h` | Struct definitions | **Yes. Top priority.** |

**Steps:**

1. **Compare the old and new protocol PDFs** with a PDF comparison tool, and list every change.
2. **`referee.h` (do this first):** Update the struct names and contents to match the new protocol.
    - In the protocol, a size of **1 = 8 bits**, so use `uint8_t`.
    - Use bit fields like `: 4` to split an 8-bit value.
    
    **Example: `0x0001` game status**
    
    | Byte offset | Size (bytes) | Contents |
    | --- | --- | --- |
    | 0 | 1 | **bits 0–3:** game type (1 = RMUC, 2 = University single events, 3 = ICRA AI Challenge, 4 = RMUL 3v3, 5 = RMUL Infantry)<br>**bits 4–7:** current stage (0 = not started, 1 = preparation, 2 = 15 s referee self-check, 3 = 5 s countdown, 4 = in match, 5 = settlement) |
    | 1 | 2 | Time remaining in current stage (s) |
    | 3 | 8 | UNIX time (valid once the robot connects to the referee NTP server) |
    
    ```c
    typedef __packed struct
    {
        uint8_t  game_type : 4;     // byte 0, bits 0-3
        uint8_t  game_progress : 4; // byte 0, bits 4-7
        uint16_t stage_remain_time; // bytes 1-2
        uint64_t SyncTimeStamp;     // bytes 3-10
    } game_status_t;
    ```
    
    How to read it: size **1** = `uint8_t`, size **2** = `uint16_t`, size **8** = `uint64_t`. The 1-byte field holds two 4-bit values, so it's split with `: 4` twice. The struct must be **packed** so there's no padding between fields.
    
    - The values for `robot_id_t` and `operator_robot_id_t` are in **Appendix 2** of the protocol.
3. **`rm_referee.h`:** Extend `referee_cmd_id_t`.
    - Naming rule: all **UPPERCASE**, replace `_t` with `_CMD_ID`.
    - Example: `game_status_t` → `GAME_STATUS_CMD_ID`
4. **`referee.c`:** Read the whole file first to understand how data is received and distributed. Then:
    - Update `init_referee_struct_data` and `referee_data_solve` so the new data is received correctly.
    - Check the `get` functions. Changing them doesn't affect receiving, **but if any bit meanings changed, update every place that uses that `get` function.**
    - Example: `event_data_t`'s bit meanings changed in RMNA 2026. It's read by `get_event_data_ptr()`, which `miniPC_msg.c` uses for zone occupancy, so `miniPC_msg.c` had to be updated too.
5. **Check every place that uses changed data** (such as `miniPC_msg.c`) so nothing receives garbled data.
6. **Coordinate with the software team** on anything that affects MiniPC ↔ C-board communication (their `serial_com`).
7. **Simulate with the competition engine** to verify the data.

### 15.4 Competition rule changes

Rules are revised **every year**. Update embedded code to match, such as **heat calculations, energy calculations, and violation penalties**. Think about whether it's possible or worth it to bend a rule within reasonable limits (for example, slightly exceeding heat or initial velocity) for an advantage.

> 📜 *A trip down memory lane:* In the “base” era, going over the heat limit only cost part of the robot's health. So we had a **"suicide mode"**: go over the heat limit to fire as many large projectiles as possible before losing all health, to turn the score around or even end the match instantly.
> 
- [x]  Review rule changes
- [ ]  New launch speed and heat controls (see **Britny's RMUL 2026 Rule Manual Differences**)
- [x]  New power control

---

## 16. Adding New Hardware (Example: Radar Mount)

In Season 26, the minimum size rule for the Sentry matched the Infantry, so we added code to **raise and lower a radar mount**. Use this as a template for adding new hardware.

**Approach:**

1. **Put it with similar logic.** The radar mount works like the **ammunition hatch servo**, so it lives in the same file: `robot_cmd.c`.
2. **Register the motors.** It uses **two 2006 motors**. Follow the registration code for the feeder motors.
3. **Set directions.** There's one motor on each side, so they spin in **opposite** directions. Set this when registering them.
4. **Control logic.** It's raised and lowered with the remote controller after power-up, like the ammunition hatch. Follow the `servoControl` logic.
5. **Auto-stop at the limit.** Reuse the **stall detection** logic from the feeder motor.
6. **Make it configurable.** Add macros in `robot_def.h` so each robot can enable or configure it.

**Result:**

- `robot_cmd.c`: `lidar_stand_config`, `LidarStandControl`
- `robot_def.h`: under `// Robot Accessories`

```c
// Robot accessories
// #define SERVO
// #define LIDAR_STAND
```

To enable an accessory on a robot, **uncomment** its line. Both are off by default.

Other hardware updates follow each season's requirements.

---

## 17. Troubleshooting and Known Issues

### 17.1 ME review checklist

Attend the **ME final review meeting** and check:

- [ ]  Cable **lengths** are right
- [ ]  Cables are **easy to connect**
- [ ]  Cables are **protected**, not easily pulled, and not at risk from moving parts
- [ ]  New hardware that needs embedded control is **practical to implement in code**
- [ ]  Yaw and pitch motor **mounting orientation** is easy to debug

### 17.2 Quick checks

| Symptom | First thing to check |
| --- | --- |
| A function stopped working | **OLED:** is that hardware online? |
| DM motor isn't at zero after power-on (code is correct) | **DM firmware version.** V17 fixes zero-point issues ([§5.1](#51-check-the-firmware-version-do-this-first)) |
| Yaw zero point drifts | **PMAX** not set for yaw ([§5.5](#55-set-pmax--vmax--tmax)) |
| Chassis locked, buffer energy negative | Main controller firmware **12.0.0.7**. Use **12.0.0.3** |
| No video feed | Video transmission and operator terminal **major versions** don't match |
| Code won't run while debugging | More than **5 breakpoints** |
| UI lags | Too much UI data or updates too often |
| Pitch pushes down and locks at the hard limit | **Resolved:** caused by the motor PID's **max out** setting |

### 17.3 C-board freezes (open issue)

**Symptoms:**

- The color-changing LED stops changing
- The values in the OLED's top-left and top-right corners stop changing

**Troubleshooting steps:**

1. Disable the watchdog:
    
    ![Disable watchdog](https://cdn.nlark.com/yuque/0/2025/png/21689302/1764314603737-dfa27610-ef1f-4f18-94b6-881f89c43a0a.png)
2. Disable watchdog initialization:
    
    ![Disable watchdog init](https://cdn.nlark.com/yuque/0/2025/png/21689302/1764314628982-3c3f2dfb-cc62-4abd-8e1d-a34587bab5f4.png)
- [ ]  Root cause still under investigation

### 17.4 Yaw doesn't initialize facing forward after a DM firmware upgrade (open issue)

- The C-board debug logs show the yaw zero point shifted. The DM debugger shows the motor's zero point is **still aligned forward**, so it's **not** gear skipping or a mechanical issue.
- **Current workaround:**
    1. Reflash the C-board firmware
    2. Unplug and replug the motor's **XT30** connector
- Suspected cause: C-board code. Reflashing the same program sometimes fixes it, but not always.
- [ ]  Root cause still under investigation

### 17.5 Other tips

- **Motor IDs and orientation:** When installing motors, double-check the IDs and orientation, **especially the chassis motors!** Mistakes mean rework.
- **Sentry video:** The Sentry (Shao Bing) video transmission doesn't show a live feed. You can switch the main controller to **Infantry** mode so ME can use the RM operator client to debug trajectory.
- **Swapping `robot_cmd.c`:** The biggest difference between the Infantry and Sentry code is `robot_cmd.c`. You can flash the Sentry version onto an Infantry (and vice versa). This helps software debugging: with the Sentry version, putting the **right joystick in the middle enables auto-aim**, without opening the operator terminal and pressing the right mouse button.
- **2026 client mouse:** The up/down mouse direction seemed reversed. It might be a bug, or it might need to be reversed at the embedded level.

### 17.6 Old video transmission: status light is green/yellow but there's no video

From the DJI video transmission troubleshooting guide. Try these in order:

1. **Receiver network adapter IP is wrong**
    1. On the computer, open **Device Manager → Network adapters**.
    2. Look for **"Remote NDIS based Internet Sharing Device."** If it isn't there, make sure the receiver's USB is plugged in and the computer is online. The driver installs automatically.
    3. Check that the virtual adapter's IP is **192.168.42.105**. If not, set it manually.
    4. Restart the video transmission **receiver**.
2. **Firewall is blocking the video:** Turn off the firewall.
3. **One-time startup glitch:** Restart the **receiver** (plug in the USB cable first, then the power adapter). If there's still no video, restart the **transmitter** too.

### 17.7 Setting up RFID cards

1. Change the language of the **referee system on the robot** to **Chinese**. The RFID setup options only show up in Chinese.
2. Open the RFID settings and assign the card to a **field location**.
3. The first list only shows **RMUC** locations (such as highland or ramp zones). Scroll further to find the **League (联盟赛)** options, such as the red repair zone or the center point.
4. Choose the location that matches where the card will be used, then test it.

---

## 18. Glossary

| Term | Meaning |
| --- | --- |
| **C-board** | RoboMaster Development Board Type C (main controller) |
| **DM motor** | DAMIAO motor (e.g., DM 4310) |
| **RM / DJI motor** | DJI M3508, M2006, etc. |
| **ESC** | Electronic speed controller (C620, C610) |
| **ME** | Mechanical branch |
| **Infantry (INF)** | Standard robot |
| **Sentry (Shao Bing)** | Autonomous sentry robot |
| **Hero** | Large-projectile (42 mm) robot |
| **Engineer** | Robot added in this season's rules |
| **Outer loop** | The loop the motor's target is set on (angle loop normally, speed loop during speed-loop tuning) |
| **Zero point** | Encoder position treated as 0° (pitch horizontal, yaw facing forward) |
| **Feedforward** | Pre-calculated output added on top of PID (e.g., gravity compensation) |
| **MANUAL / AUTO PID** | PID used when the operator drives / when auto-aim controls |
| **Kill switch** | Wireless relay that cuts power to the robot |

---

*Just do what you want to do.* 🤖
