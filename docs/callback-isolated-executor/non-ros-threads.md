# Non-ROS Threads

The thread configurator can also manage scheduling for threads that are not part of any ROS 2 executor — for example, sensor driver threads or custom worker threads.

Each non-ROS thread reports its TID and a logical name to the configurator daemon, which then applies the scheduling parameters defined in the YAML configuration. Clients are available for both C++ and Rust.

## C++

Use `agnocast_cie_thread_configurator::spawn_non_ros2_thread` instead of `std::thread` to create the thread. This function automatically reports the thread's ID to the configurator so that scheduling parameters can be applied.

```cpp
#include "agnocast_cie_thread_configurator/cie_thread_configurator.hpp"

// Instead of:
//   std::thread t(my_worker_function, arg1, arg2);

// Use:
std::thread t = agnocast_cie_thread_configurator::spawn_non_ros2_thread(
  "my_worker",        // unique thread name (must match YAML config)
  my_worker_function,
  arg1, arg2);
```

The `thread_name` must be unique among all threads managed by the configurator, and must match the `name` field in the YAML configuration.

### CMake

Add `agnocast_cie_thread_configurator` as a dependency:

```cmake
find_package(agnocast_cie_thread_configurator REQUIRED)
ament_target_dependencies(your_target agnocast_cie_thread_configurator)
```

## Rust

A pure-Rust client crate (`agnocast_cie_thread_configurator`) implements the same protocol for Rust programs. It exposes two entry points:

- `spawn_non_ros2_thread(thread_name, closure)` — spawns a thread that reports itself to the configurator, then runs the closure; returns its `JoinHandle<T>`. This is the Rust counterpart of the C++ function.
- `report_current_thread(thread_name)` — reports the calling thread. Use this for threads that are not created via `spawn_non_ros2_thread`, such as the `main` thread.

```rust
use agnocast_cie_thread_configurator::{report_current_thread, spawn_non_ros2_thread};

fn main() {
    // Report a thread created outside spawn_non_ros2_thread (e.g. main).
    report_current_thread("main_thread");

    // Spawn a managed worker thread.
    let handle = spawn_non_ros2_thread("my_worker", || {
        // worker body
    });

    handle.join().unwrap();
}
```

As with C++, `thread_name` must be unique among all threads managed by the configurator, and must match the `name` field in the YAML configuration.

### Cargo

The crate lives in the Agnocast repository under `src/agnocast_cie_thread_configurator/rust/` and is built standalone with cargo (it has no `package.xml`, so colcon ignores it). Add it as a git dependency:

```toml
[dependencies]
agnocast_cie_thread_configurator = { git = "https://github.com/autowarefoundation/agnocast.git" }
```

The crate is dependency-light (only `libc` and `log`) and links against neither Agnocast, ROS 2, nor DDS, so it can be statically linked — for example against `x86_64-unknown-linux-musl`. Reporting is best-effort: if the configurator daemon is not running, the failure is logged through the `log` crate and the thread runs normally.

### Toggling Agnocast on and off

The crate performs no gating of its own — it always reports when called, and whether to call it at all is decided entirely on the consumer side. The recommended pattern is to make it an optional dependency behind a cargo feature that is **off by default**, so the program can be built either with or without Agnocast (for example, to ship a statically-linked binary in an environment where Agnocast is not installed):

```toml
[dependencies]
agnocast_cie_thread_configurator = { git = "https://github.com/autowarefoundation/agnocast.git", optional = true }

[features]
default = []
agnocast = ["dep:agnocast_cie_thread_configurator"]
```

Gate the calls on that feature so they compile out completely when it is disabled:

```rust
#[cfg(feature = "agnocast")]
use agnocast_cie_thread_configurator::report_current_thread;

fn main() {
    #[cfg(feature = "agnocast")]
    report_current_thread("main_thread");

    // ... rest of the program runs identically with or without the feature ...
}
```

Build with Agnocast off (default) or on:

```bash
cargo build                       # Agnocast off: crate is not compiled in
cargo build --features agnocast   # Agnocast on: threads are reported
```

When the feature is off, the crate is neither compiled nor linked. When it is on, reporting still degrades gracefully if the configurator daemon is absent (see above).

## YAML Configuration

Configure non-ROS threads under the `non_ros_threads` section:

```yaml
non_ros_threads:
  - name: my_worker
    policy: SCHED_FIFO
    priority: 85
    affinity: [3]
```

The fields are the same as `callback_groups` except `name` is used instead of `id`. All scheduling policies including `SCHED_DEADLINE` are supported. See the [YAML Specification](yaml-specification.md#non_ros_threads) for details.

## Prerun Mode

Non-ROS threads are also discovered by the prerun mode and included in the generated `template.yaml`.
