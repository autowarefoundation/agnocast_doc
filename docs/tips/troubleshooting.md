# Troubleshooting

## Cleaning up shared memory

Agnocast runs a daemon process that automatically cleans up shared memory, even when Agnocast participant processes crash. If the daemon itself is killed, these resources persist until the next Agnocast process starts, which spawns a replacement that unlinks them. To clean them up without waiting for that:

```bash
rm -f /dev/shm/agnocast@*
```
