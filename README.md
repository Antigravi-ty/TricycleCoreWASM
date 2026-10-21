![image](https://user-images.githubusercontent.com/36944229/219303954-7267bce1-b7c5-4f15-881c-b9545512e65b.png)

**A C++ library for simulating Rocket League games at maximum efficiency**

RocketSim is a complete simulation of Rocket League's gameplay logic and physics that is completely standalone.
RocketSim supports the game modes: Soccar, Hoops, Dropshot, Heatseeker, and Snowday.

# Speed
RocketSim is designed to run extremely fast, even when complex collisions and suspension calculations are happening every tick.
On an average PC running a single thread of RocketSim with two cars, RocketSim can simulate around 20 minutes of game time every second.
This means that with 12 threads running RocketSim, you can simulate around 10 days of game time every minute!

# Accuracy
RocketSim is not a perfectly accurate replication of Rocket League, but is close enough for most applications (such as training ML bots).
RocketSim is accurate enough to:
- *Train machine learning bots to SSL level (and probably beyond)*
- *Simulate different shots on the ball at different angles to find the best input combination*
- *Simulate air control to find the optimal orientation input*
- *Simulate pinches*

However, the tiny errors will accumulate over time, so RocketSim is best suited for simulation with consistent feedback.

## Installation
- Clone this repo and build it
- Use https://github.com/ZealanL/RLArenaCollisionDumper to dump all of Rocket League's arena collision meshes
- Move those assets into RocketSim's executing directory

## Documentation
Documentation is available at: https://zealanl.github.io/RocketSimDocs/

## Bindings
If you don't want to work in C++, here are some (unofficial) bindings written in other languages:
- **Python**: https://github.com/mtheall/RocketSim by `mtheall`
- **Rust**: https://github.com/VirxEC/rocketsim-rs by `VirxEC`

Official Python bindings are currently in the works.

## Performance Details
RocketSim already heavily outperforms the speed of Rocket League's physics tick step without optimization.

Version performance comparison:
```
OS: Windows 10 (Process Priority = Normal)
CPU: Intel i5-11400 @ 2.60GHz
Ram Speed: 3200MZ
Compiler: MSVC 14.16
=================================
Arena: Default (Soccar)
Cars: 2 on each team (2v2)
Inputs: Randomly pre-generated, changed every 2-60 ticks for each car
=================================
Single-thread performance (calculated using average CPU cycles per tick on the RocketSim thread) (1M ticks simulated):
v1.0.0 = 30,334tps
v1.1.0 = 48,191tps
v1.2.0 = 50,763tps
v2.0.0 = ~50,000tps
v2.1.0 = 114,481tps
```

## Issues & PRs
Feel free to make issues and pull requests if you encounter any issues!

You can also contact me on Discord if you have questions, my username is `zealanl`.

## Legal Notice
RocketSim was written to replicate Rocket League's game logic, but does not actually contain any code from the game.
To Epic Games/Psyonix: If any of you guys have an issue with this, let me know on Discord and we can resolve it.


---

# TricycleCoreWASM (Three-Wheeler & Throwback Stadium Extensions)

This repository contains the modified RocketSim C++ & Emscripten WASM bridge (`ReWASM_EDIT` base) supporting:

1. **Three-Wheeler Mode ("三蹦子" / `CAR_CONFIG_PSYCLOPS` / `Body_Topple` / `Confidential_ThirdWheel_Test`)**
2. **Throwback Stadium (`ThrowbackStadium_P.upk` non-standard arena with standalone goals, 44 boost pads, and open back-of-goal area)**

## 1. Three-Wheeler Mode (`CAR_CONFIG_PSYCLOPS`) Changes
- **`wasm/RocketSimBridge.cpp`**:
  - Added `case 6: return CAR_CONFIG_PSYCLOPS;` to `getCarConfigForIndex(int hitboxIndex)` so `HITBOX_PRESETS.PSYCLOPS = 6` in JavaScript selects the native 3-wheeled Octane/Topple configuration (`threeWheels = true`, front axle narrowed to `Y = ±5.0 uu`, front rest length `33.0 uu`, back rest length `31.3 uu`, `STEER_ANGLE_FROM_SPEED_CURVE_THREEWHEEL`, `LAT_FRICTION_CURVE_THREEWHEEL`, and `stickyForceScale = 0`).
  - Added C-ABI exports `_physics_setCarThreeWheels(int carId, int enabled)` and `_physics_getCarThreeWheels(int carId)` to toggle `threeWheels` on any car at runtime.
- **`wasm/compile.py`**:
  - Added `_physics_setCarThreeWheels`, `_physics_getCarThreeWheels`, `_physics_setArenaPreset`, `_physics_getArenaPreset`, and `_physics_setGoalVolumeBox` to `EXPORTED_FUNCTIONS`.

## 2. Throwback Stadium (`ThrowbackStadium_P.upk`) Physics Changes
- **`src/RLConst.h` (`namespace ThrowbackStadium`)**:
  - Added exact coordinates extracted from `ThrowbackStadium_P.upk`:
    - **GoalVolume 3D AABB (`GoalVolume_TA_0` & `GoalVolume_TA_1`)**: `X ∈ [-960.0, +960.0]`, `|Y| ∈ [4909.31, 5757.31]`, `Z ∈ [-64.0, +648.0]` (back wall is at `Y = ±6912.0`).
    - **Boost Pads**: 8 Big Boost Pills (`BOOST_LOCATIONS_AMOUNT_BIG = 8`, including `(0, ±6512, 72)` behind the goals) and 36 Small Boost Pads (`BOOST_LOCATIONS_AMOUNT_SMALL = 36`, including 8 pads behind the goals).
    - **Kickoff Spawns & Respawns**: 5 kickoff locations (`(±2048, -2560)`, `(±256, -3840)`, `(0, -4608)`) and 4 backline respawn locations (`(±2688, -4608)`, `(±2304, -4608)`).
- **`src/Sim/MutatorConfig/MutatorConfig.h` & `src/Sim/Arena/Arena.cpp` (`Arena::IsBallScored()`)**:
  - Added `useThrowbackGoalVolume` and 3D GoalVolume bounding box parameters (`goalVolumeMinX/MaxX`, `goalVolumeMinY/MaxY`, `goalVolumeMinZ/MaxZ`).
  - When `useThrowbackGoalVolume == true`, `Arena::IsBallScored()` checks whether the ball enters the 3D `GoalVolume_TA` box (`abs(pos.y) >= minY + r && abs(pos.y) <= maxY - r && abs(pos.x) <= maxX - r && pos.z >= minZ && pos.z <= maxZ - r`), allowing the ball and cars to freely pass behind the goal (`|Y| > 5757.31` up to `6912`) without false scoring.
- **`src/Sim/Arena/ArenaConfig/ArenaConfig.h` & `src/Sim/Arena/Arena.cpp` (`Arena::_SetupArenaCollisionShapes()`)**:
  - Added `disableHardcodedSideWalls` flag to skip adding the hardcoded `X = ±4096` and `Z = 2048` `btStaticPlaneShape` planes when loading self-contained custom arena `.cmf` collision meshes (such as `ThrowbackStadium`).
  - Added `useThrowbackBoostPads` and `useThrowbackSpawns` support in `Arena::ResetToRandomKickoff()` and `Arena::RegisterGoalScore()`.
- **`wasm/RocketSimBridge.cpp`**:
  - Added `_physics_setArenaPreset(int preset)` (`0` = Standard Soccar, `1` = Throwback Stadium) to be called before `_physics_init(...)`, which automatically configures `disableHardcodedSideWalls`, `useThrowbackBoostPads`, `useThrowbackSpawns`, and `useThrowbackGoalVolume`.
