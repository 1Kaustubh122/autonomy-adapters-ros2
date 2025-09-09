# autonomy-adapters-ros2

**Status:** Under development.  

This repository provides deterministic ROS 2 ↔ ICTK adapters.  
Controllers run inside a real-time island with **zero heap in the hot path**, while ROS 2/DDS I/O and logging are handled in non-RT threads.  

---

## Scope

- Bridge ROS 2 topics to ICTK `UpdateContext` / `Result` at a fixed rate.  
- Run control logic in a **pinned SCHED_FIFO thread** using `CLOCK_MONOTONIC_RAW`.  
- All RT paths use **bounded messages, fixed arrays, and preallocated buffers**.  
- Non-RT side manages ROS 2 publishers/subscribers, parameter updates, diagnostics, and bagging.  
- Loaned messages are optional; if not supported, a ringbuffer publisher path is always valid.  

---

## Repository layout

```

autonomy-adapters-ros2/
ictk_ros2_core/         # RT utilities (no rclcpp, -fno-exceptions/-fno-rtti)
ictk_ros2_msgs/         # IDL fixed/bounded messages
ictk_ros2_pid/          # PID Lifecycle component (wraps ICTK::PIDCore)
ictk_ros2_diagnostics/  # non-RT health publishers / recorder helpers
examples/               # launch files for demo graphs
config/                 # DDS XML profiles, CPU pinning scripts
test/                   # jitter, zero-alloc, pool-lock, roundtrip, param-swap

````

---

## Messages

Defined in `ictk_ros2_msgs` using **IDL with fixed bounds**:

- `PlantState` – timestamp, outputs `y[]`, estimates `xhat[]`, validity bits.  
- `Setpoint` – reference vector `r[]`, preview horizon length.  
- `ControlCommand` – timestamp, command vector `u[]`.  
- `ControllerHealth` – watchdog/deadline counts, saturation %, rate/jerk hits, flags, last limit magnitudes.  

All RT messages are fixed-size (bounded sequences, explicit lengths, no strings).  
Non-RT messages (`ControllerHealth`) may include richer fields.  

---

## Real-time architecture

- **RT thread:**  
  - Pinned SCHED_FIFO, single waitset timer.  
  - Reads latest-value cache snapshots.  
  - Runs ICTK controller.  
  - Writes a fixed frame into an SPSC ringbuffer.  

- **Subscriptions (non-RT):**  
  - ROS 2 subs update latest-value caches (lock-free, seq fenced).  

- **Publisher thread (non-RT):**  
  - Reads frames from ringbuffer.  
  - Publishes `/command` (loaned if available, else normal).  
  - Publishes `/health` at reduced rate.  

- **Parameters:**  
  - Validated at configure.  
  - Runtime reconfig via epoch-swap at tick boundary.  
  - At most one apply per N ticks (bounded jitter).  

---

## QoS

- `/plant_state`: RELIABLE, KEEP_LAST(2), deadline = `dt`.  
- `/setpoint`: RELIABLE, KEEP_LAST(5).  
- `/command`: BEST_EFFORT, KEEP_LAST(1), deadline = `dt`.  
- `/health`: RELIABLE, KEEP_LAST(10).  

---

## Build

```bash
source /opt/ros/humble/setup.bash   # or iron
git clone <repo_url> autonomy-adapters-ros2
cd autonomy-adapters-ros2

colcon build --symlink-install 
  --cmake-args 
    -DADAPTER_RT_SCHED=ON 
    -DADAPTER_FORCE_RINGBUF_PUB=OFF 
    -DADAPTER_N_MAX=8
````

**Note:**
Grant SCHED_FIFO privileges (example: `sudo setcap cap_sys_nice+ep $(which component_container_mt)`), or the adapter falls back to SCHED_OTHER (pinned) with a health warning.

---

## Tests

```bash
colcon test --event-handlers console_cohesion+
```

Covers:

* RT jitter bounds.
* Zero-allocation in RT loop (interposer).
* DDS pool stability after 10k publishes.
* Round-trip PlantState → ICTK → ControlCommand.
* Param swap jitter ≤ 5% of `dt`.

---

## Roadmap

* **PR-A:** Messages, RT core, PID component skeleton, tests, XML configs.
* **PR-B:** Compliance logger (`raw.binlog`, `metadata.json`).
* **PR-C:** Demo graphs (planner, safety stop).
* **PR-D:** Bridges for SFTK and later RLTK.

---

## License

Apache-2.0


