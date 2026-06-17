# Integration Guide

## Overview

1. **Switch the executor** in your application code or launch file
2. **Set up the thread configurator** (one-time system configuration)
3. **Generate a YAML template** by running the configurator in the prerun mode
4. **Edit the YAML** to assign scheduling policies, priorities, and CPU affinities
5. **Launch the configurator** with your config file, then start your application
6. *(Optional)* **Tune at runtime** by editing the YAML and re-applying it through the `reapply_config` reload service, without restarting

See the [Tutorial](tutorial.md) for a concrete walkthrough with a sample application.

## Step 1: Switch the Executor

### Nodes with a `main` function

Replace the executor:

**Before:**

```cpp
rclcpp::executors::SingleThreadedExecutor executor;
```

**After:**

```cpp
agnocast::CallbackIsolatedAgnocastExecutor executor;
```

### Composable Nodes

Use `agnocast_component_container_cie` in your launch file:

```xml
<node_container pkg="agnocast_components" exec="agnocast_component_container_cie"
                name="my_container" namespace="" output="screen">
    <!-- LD_PRELOAD is only needed when using Agnocast pub/sub -->

    <composable_node pkg="my_package" plugin="MyNode"
                     name="my_node" namespace="">
    </composable_node>
</node_container>
```

Or via CMake registration:

```cmake
agnocast_components_register_node(
  my_component
  PLUGIN "MyNode"
  EXECUTABLE my_node
  EXECUTOR CallbackIsolatedAgnocastExecutor
)
```

For multiple ROS domains, launch one container process per domain and set
`ROS_DOMAIN_ID` on each container process:

```xml
<!-- Domain 0 (default) -->
<node_container pkg="agnocast_components" exec="agnocast_component_container_cie"
                name="my_container_domain_0" namespace="" output="screen">
    <composable_node pkg="my_package" plugin="MyNode" name="my_node" namespace="" />
</node_container>

<!-- Domain 1 -->
<node_container pkg="agnocast_components" exec="agnocast_component_container_cie"
                name="my_container_domain_1" namespace="" output="screen">
    <env name="ROS_DOMAIN_ID" value="1" />
    <composable_node pkg="my_package" plugin="MyNode" name="my_node" namespace="" />
</node_container>
```

!!! warning
    Due to [autowarefoundation/agnocast#1263](https://github.com/autowarefoundation/agnocast/issues/1263), `CallbackIsolatedAgnocastExecutor` and `agnocast_component_container_cie` assume that all callback groups containing Agnocast callbacks are created before they are associated with the Executor by calling `add_node()` or `add_callback_group()`. If an Agnocast callback is added to a callback group after the callback group is associated with the Executor, the wrong executor type may be selected and the callback will not execute.

## Step 2: Set Up the Thread Configurator

### Grant capabilities

```bash
sudo setcap cap_sys_nice=eip $(readlink -f $(ros2 pkg prefix agnocast_cie_thread_configurator)/lib/agnocast_cie_thread_configurator/thread_configurator_node)
```

!!! note
    `setcap` does not work on symlinks. `readlink -f` resolves to the actual binary, which is needed when using `colcon build --symlink-install`.

### Configure library paths

After `setcap`, the dynamic linker ignores `LD_LIBRARY_PATH` for security. Register the paths explicitly:

Create `/etc/ld.so.conf.d/agnocast-cie.conf` with all directories that contain libraries needed by the thread configurator. For example:

```bash
echo "/opt/ros/$ROS_DISTRO/lib
/opt/ros/$ROS_DISTRO/lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH)
$(ros2 pkg prefix agnocast_cie_config_msgs)/lib" | sudo tee /etc/ld.so.conf.d/agnocast-cie.conf
```

Apply:

```bash
sudo ldconfig
```

### Kernel boot parameter (SCHED_DEADLINE only)

If using `SCHED_DEADLINE`, CPU affinity requires cgroup v1:

```text
GRUB_CMDLINE_LINUX_DEFAULT="... systemd.unified_cgroup_hierarchy=0 ..."
```

```bash
sudo update-grub
sudo reboot
```

!!! note
    `SCHED_DEADLINE` cannot be set via capabilities alone — the configurator must be run as **root**.

## Step 3: Generate a YAML Template

Start the prerun node **before** launching your application:

```bash
# Terminal 1: start the prerun node first
ros2 launch agnocast_cie_thread_configurator thread_configurator.launch.xml prerun:=true

# Terminal 2: then launch your application
ros2 launch your_package your_launch.xml
```

Once all CallbackGroups are discovered, press Ctrl+C in the prerun terminal. A `template.yaml` is created in the current directory.

## Step 4: Edit the YAML

Edit the template to assign scheduling parameters. See the [YAML Specification](yaml-specification.md) for all options.

For callback groups that don't need configuration, delete the entry or leave the defaults.

## Step 5: Launch with Configuration

Start the configurator **before** the target application:

```bash
ros2 launch agnocast_cie_thread_configurator thread_configurator.launch.xml config_file:=/path/to/config.yaml
```

Then launch your application. The configurator applies scheduling parameters as CallbackGroups register.

The configurator keeps running after all configurations have been applied. If the target application restarts, the configurator automatically re-applies configurations to newly discovered threads.

!!! note
    The configurator automatically subscribes to all ROS domains referenced by `domain_id` in the YAML configuration. For the prerun mode, use the `domains` parameter to specify which domains to discover.

## Step 6: Tune the Configuration with the Reload Service (Optional)

Finding the right scheduling parameters is usually iterative. Instead of restarting the configurator and your application on every change, you can edit the YAML and re-apply it at runtime through the `reapply_config` service. This lets you tune priorities, CPU affinities, and policies in place while everything keeps running.

After editing the file you passed as `config_file` (edit that same path in place), call the service:

```bash
ros2 service call /thread_configurator_node/reapply_config \
  agnocast_cie_config_msgs/srv/ReapplyConfig "{}"
```

The configurator re-reads the file at its `config_file` parameter and re-applies the scheduling parameters to every thread that has already announced itself. A typical tuning loop is therefore: edit the YAML, call the service, verify with `chrt` / `taskset`, and repeat.

- **Added entries** take effect as soon as the corresponding thread announces itself.
- **Removed entries** drop out of the configurator's in-memory state. Scheduling already applied to a running thread is not reverted.
- **`hardware_info` and `rt_throttling` are not re-evaluated.** Changing those sections requires restarting the configurator.

If the YAML fails to parse or a per-entry validation check fails, the request is rejected, no state is changed, `success` is `false`, and `error_message` explains why.

The response reports the outcome per thread:

| Field | Meaning |
|-------|---------|
| `success` | `true` if parsing and validation passed |
| `error_message` | Failure reason (empty on success) |
| `applied_callback_groups` / `applied_non_ros_threads` | Scheduling syscalls succeeded |
| `failed_callback_groups` / `failed_non_ros_threads` | Syscall failed (see the configurator log for details) |
| `skipped_callback_groups` / `skipped_non_ros_threads` | Thread not yet announced; will apply on the next announcement |
