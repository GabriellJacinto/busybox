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
| `vc_gateway_uplink_port` | Port for ECU gateway uplink (multicast traffic from leaves). | Integer | `300` |
| `vc_gateway_downlink_port` | Port for ECU gateway downlink (unicast traffic to leaves). | Integer | `0` (disabled) |
| `vc_ecu_port` | Fixed port used by ECU component. If not set, derived from `vc_base_port + 100`. | Integer | empty (auto-calculated) |
| `vc_port_stride` | Port offset multiplier for leaf components (used in port calculation). | Integer >= 1 | `1` |
| `vc_shm_proc_count` | Number of participants in shared memory region. Must match launched process count. | Integer >= 1 | `4` |
| `vc_shm_buffers` | Buffer slots in shared-memory region. | Integer >= 1 | `8` |
| `vc_shm_key` | System V shared memory key. | Integer | `74560` |
| `vc_sem_key` | System V semaphore key. | Integer | `74561` |
| `vc_period_ecu` | ECU send/update period in ms. | Integer >= 0 | `500` |
| `vc_period_radar` | Radar send/update period in ms. | Integer >= 0 | `200` |
| `vc_period_lidar` | Lidar send/update period in ms. | Integer >= 0 | `200` |
| `vc_period_gps` | GPS send/update period in ms. | Integer >= 0 | `500` |
| `vc_measure_rtt` | Enable RTT (round-trip-time) measurement probes. If set to `1`, vehicle_component will send RTT probes in parallel with normal traffic. | `0`, `1` | `0` |

## Port mapping used by init

Given `vc_base_port = B` and `vc_port_stride = S`:
- `radar` uses `B + 0 * S`
- `lidar` uses `B + 1 * S`
- `gps` uses `B + 2 * S`
- `ecu` uses `vc_ecu_port` if set, otherwise `B + 100`

Example with `vc_base_port=200` and `vc_port_stride=1`:
- radar: 200, lidar: 201, gps: 202, ecu: 300

## Channel resolution used by init

1. If `vc_channel` is set:
- all components use `vc_channel`

2. If `vc_channel` is empty:
- `ecu` uses `vc_gateway_channel`
- `radar`, `lidar`, `gps` use `vc_leaf_channel`

## Example `-append`

### Baseline (shared memory + network hybrid)

```text
root=/dev/ram rw console=ttyS0 hostname=vehicle1 \
vc_id=1 vc_timeout=30 vc_quadrant=0 vc_quadrant_change_ms=5000 \
vc_base_port=200 vc_port_stride=1 \
vc_gateway_component=ecu vc_gateway_channel=both vc_leaf_channel=shm \
vc_gateway_uplink_port=300 vc_gateway_downlink_port=0 \
vc_shm_proc_count=4 vc_shm_buffers=8 vc_shm_key=74560 vc_sem_key=74561 \
vc_period_ecu=500 vc_period_radar=200 vc_period_lidar=200 vc_period_gps=500 \
vc_measure_rtt=0
```

### With RTT enabled (for latency measurement)

```text
root=/dev/ram rw console=ttyS0 hostname=vehicle1 \
vc_id=1 vc_timeout=30 vc_quadrant=0 vc_quadrant_change_ms=5000 \
vc_base_port=200 vc_port_stride=1 \
vc_gateway_component=ecu vc_gateway_channel=both vc_leaf_channel=shm \
vc_gateway_uplink_port=300 vc_gateway_downlink_port=0 \
vc_shm_proc_count=4 vc_shm_buffers=8 vc_shm_key=74560 vc_sem_key=74561 \
vc_period_ecu=500 vc_period_radar=200 vc_period_lidar=200 vc_period_gps=500 \
vc_measure_rtt=1
```

## Integration with Batch Analysis Scripts

These init parameters are typically configured through the Makefile scenario definitions (e.g., `Makefile` target `sim-net-2`, `sim-gateway-2`, etc.), which embed them in QEMU `-append` directives.

### Batch Execution Workflow

To run repeated simulations and collect latency metrics:

```bash
python3 scripts/run_simulation_batch.py \
  --scenario-name sim-net-2-rtt \
  --scenario-cmd "make sim-net-2 POST_SIM=0" \
  --runs 10 \
  --timeout 300 \
  --enable-rtt
```

This invokes `make sim-net-2 POST_SIM=0` 10 times, where each invocation:
1. Cross-compiles vehicle_component with RTT instrumentation (when `--enable-rtt`)
2. Launches N QEMU VMs (each with unique `hostname=vehicle*`)
3. Passes init parameters via `-append` (defined in Makefile targets)
4. Collects `logs/qemu/vehicle*/latency.csv` after each run
5. Analyzes with `latency_analysis.py`, consolidating 8 latency metrics

See [scripts/README.md](../scripts/README.md) for complete analysis pipeline documentation.

### Init Parameters Impact on Analysis

| Parameter | Measurement Impact | Typical Value |
|--|--|--|
| `vc_channel` | Determines communication path (net vs shm) affects `app_tx_to_app_rx`, `nic_tx_to_nic_rx` latencies | `both` (baseline), `net` (network-only), `shm` (shared-memory-only) |
| `vc_base_port` | Port numbering; no direct latency impact but ensures isolation between vehicles | `200` (vehicle1), `300` (vehicle2), etc. |
| `vc_period_*` | Send frequency; affects throughput `tx_fps`, `rx_fps` and loss percentage | `200`–`500` ms |
| `vc_shm_buffers` | Ring buffer capacity; larger values reduce loss in high-load scenarios | `8`–`16` |
| `vc_timeout` | Max runtime; used for long-running experiments to prevent hung VMs | `0` (no limit), or `30`–`60` for controlled tests |

### Logging and Latency Events

During execution, each vehicle component logs timestamped events to `logs/qemu/vehicle${vc_id}/latency.csv`:

```
ts_ns,event,trace_id,size,port,pid,tid
1713100000001234000,APP_TX,72057600000001,64,200,1001,1002
1713100000001235000,PROTO_TX,72057600000001,84,200,1001,1002
1713100000001236000,NIC_TX,72057600000001,84,200,1001,1002
```

These events are later correlated by `latency_analysis.py` to compute:
- **Single-clock metrics** (no cross-VM bias): `app_tx_to_proto_tx`, `proto_tx_to_nic_tx`, `nic_rx_to_proto_rx`, `proto_rx_to_app_rx`, `rtt_tx_to_rtt_rx`
- **Cross-clock metrics** (subject to bias): `app_tx_to_app_rx`, `nic_tx_to_nic_rx`, `proto_tx_to_proto_rx`

### Example: Configuring for Network Latency Measurement

Makefile target for network-only baseline (no shared memory):

```makefile
sim-net-2-baseline:
	$(MAKE) install-vehicle RISC_ARCH=$(RISC_ARCH)
	$(QEMU_CMD) -append "root=/dev/ram rw console=ttyS0 hostname=vehicle1 \
	  vc_channel=net vc_base_port=200" &
	$(QEMU_CMD) -append "root=/dev/ram rw console=ttyS0 hostname=vehicle2 \
	  vc_channel=net vc_base_port=300" &
	wait
```

Then measure with:

```bash
python3 scripts/run_simulation_batch.py \
  --scenario-name sim-net-2-baseline \
  --scenario-cmd "make sim-net-2-baseline POST_SIM=0" \
  --runs 10 --timeout 300
```

Output analysis in `logs/experiments/batch_*/summary_metrics.csv` and `run_*/latency.json`.
