# Configuring the Domain Bridge

One [ROS 2 `domain_bridge`](https://github.com/ros2/domain_bridge) YAML config drives both sides: the `domain_bridge` node relays the topics it lists, and the Agnocast discovery agent reads the same file to bring up the Agnocast→ROS 2 bridge each of those topics needs. Keeping them in one file keeps the two in agreement.

## File format

```yaml
# Default direction for every topic below. Per-topic values override these.
from_domain: 1
to_domain: 2

topics:
  # Bridge /chatter from domain 1 to domain 2 (uses the defaults above).
  chatter:
    type: std_msgs/msg/String

  # Override the direction for a single topic, and rename it in the destination domain.
  image:
    type: sensor_msgs/msg/Image
    from_domain: 3
    to_domain: 4
    remap: image_raw
```

The discovery agent reads these fields, with the same meaning as in `domain_bridge`:

- The map key (`chatter`, `image`) is the topic name in the source domain. `remap` gives it a different name in the destination domain.
- `from_domain` / `to_domain` are `ROS_DOMAIN_ID` values. Give them at the top level as defaults, per topic, or both. A topic with no resolved `from_domain` / `to_domain` is skipped, and the agent logs it.
- `reversed: true` swaps `from_domain` and `to_domain` for that topic.
- `bidirectional: true` bridges the topic in both directions.

The agent ignores every other field, such as `type` and QoS. `domain_bridge` still needs `type`, so keep it in the file.

## Where the file goes

The discovery agent is `execv`'d from an application process, so it has no command line of its own: it finds the config through the default location or through an environment variable it inherits from that process.

- **Default location.** `/etc/agnocast/domain_bridge.yaml` is read, followed by every `*.yaml` in `/etc/agnocast/domain_bridge.d/` in name order.
- **`AGNOCAST_DOMAIN_BRIDGE_CONFIG`.** One path, or several separated by `:`. Export it to your application processes — setting it only where `domain_bridge` runs does not reach the agent. Setting it replaces the default location, so the drop-in directory is not read; the agent warns if the directory still holds a `*.yaml`.

Splitting the rules across files is the same merge the `domain_bridge` node performs: `topics` from every file accumulate, while `from_domain` / `to_domain` stay local to the file that sets them. A file that cannot be read is reported in the agent's log and skipped, so the rest still apply.

## Running `domain_bridge`

Pass the same files to the `domain_bridge` node:

```bash
ros2 run domain_bridge domain_bridge /etc/agnocast/domain_bridge.yaml /etc/agnocast/domain_bridge.d/*.yaml
```

Do not use `domain_bridge`'s `--from` / `--to` options. They override the domains in the file for the node only, and the discovery agent, which reads only the file, would then bring up the bridge in the wrong domain.
