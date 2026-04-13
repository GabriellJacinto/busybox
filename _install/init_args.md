# Init script arguments (vehicle VMs)

This document describes the `vc_*` kernel command-line parameters read by the init scripts:
- `guest_rootfs/init`
- `busybox/_install/init`

These parameters are passed through QEMU `-append "..."`.

## Quick behavior

- If `hostname` is `sender` or `receiver`, vehicle params are ignored.
- If `hostname` matches `vehicle[0-9]+`, the script launches 4 processes:
  - `ecu` (fixed gateway role)
  - `radar`
  - `lidar`
  - `gps`

## Parameters

| Parameter | Purpose | Values | Default |
|---|---|---|---|
| `vc_id` | Vehicle logical ID passed to all components. | Integer >= 1 | `1` |
| `vc_timeout` | Max runtime in seconds for each component (`0` means no timeout). | Integer >= 0 | `0` |
| `vc_quadrant` | Initial quadrant state for app logic. | Integer | `0` |
| `vc_quadrant_change_ms` | Period for quadrant changes in ms (`0` disables periodic changes). | Integer >= 0 | `0` |
| `vc_base_port` | Base application port; component ports are derived from this. | Integer | `200` |
| `vc_channel` | Global override channel for all processes. If set, it overrides leaf/gateway channel split. | `net`, `shm`, `both` | empty (disabled) |
| `vc_gateway_component` | Legacy knob from old design. In current architecture ECU is always the gateway. Non-`ecu` values are ignored with a warning. | `ecu` (effective) | `ecu` |
| `vc_gateway_channel` | Channel used by ECU gateway when `vc_channel` is not set. | `net`, `shm`, `both` | `both` |
| `vc_leaf_channel` | Channel used by leaf components (`radar`, `lidar`, `gps`) when `vc_channel` is not set. | `net`, `shm`, `both` | `shm` |
| `vc_shm_proc_count` | Number of participants in shared memory region. Must match launched process count. | Integer >= 1 | `4` |
| `vc_shm_buffers` | Buffer slots in shared-memory region. | Integer >= 1 | `8` |
| `vc_shm_key` | System V shared memory key. | Integer | `74560` |
| `vc_sem_key` | System V semaphore key. | Integer | `74561` |
| `vc_period_radar` | Radar send/update period in ms. | Integer >= 0 | `200` |
| `vc_period_lidar` | Lidar send/update period in ms. | Integer >= 0 | `200` |
| `vc_period_gps` | GPS send/update period in ms. | Integer >= 0 | `500` |

## Port mapping used by init

Given `vc_base_port = B`:
- `radar` uses `B + 0`
- `lidar` uses `B + 1`
- `gps` uses `B + 2`
- `ecu` uses `B + 100`

## Channel resolution used by init

1. If `vc_channel` is set:
- all components use `vc_channel`

2. If `vc_channel` is empty:
- `ecu` uses `vc_gateway_channel`
- `radar`, `lidar`, `gps` use `vc_leaf_channel`

## Example `-append`

```text
root=/dev/ram rw console=ttyS0 hostname=vehicle1 \
vc_id=1 vc_timeout=30 vc_quadrant=0 vc_quadrant_change_ms=5000 \
vc_base_port=200 vc_gateway_component=ecu vc_gateway_channel=both vc_leaf_channel=shm \
vc_shm_proc_count=4 vc_shm_buffers=8 vc_shm_key=74560 vc_sem_key=74561 \
vc_period_radar=200 vc_period_lidar=200 vc_period_gps=500
```
