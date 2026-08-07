---
title: "Monitoring a UPS with standard Linux components"
excerpt: "A small UPS monitoring pipeline built with NUT, systemd, awk, Node Exporter, Prometheus, and Grafana."
permalink: /nut-ups-monitoring/
categories:
  - monitoring
tags:
  - linux
  - nut
  - prometheus
  - grafana
  - ups
  - llm
header:
  overlay_image: https://raw.githubusercontent.com/filippobuletto/nut-ups-monitoring/main/docs/dashboard.png
  overlay_filter: rgba(0, 0, 0, 0.5)
toc: true
toc_sticky: true
---

A UPS can keep equipment running through a power cut or unstable mains power.
Its battery health, estimated runtime, load, and operating state determine how
much protection it can provide, so they are worth monitoring.

`nut-ups-monitoring` makes those signals visible in Prometheus and Grafana
without adding another long-running exporter or a custom application.

The UPS, or uninterruptible power supply, is the hardware that provides
temporary battery power when mains power fails or becomes unstable. Network UPS
Tools (NUT) communicates with that hardware, and its `upsc` command prints the
values reported by the device. `systemd`, the service manager used by many
Linux distributions, runs the collection job every minute and keeps its logs.

The job uses `awk`, a standard text-processing tool, to turn NUT's output into
the text format that Prometheus understands. Node Exporter is a Prometheus
exporter for host-level metrics; its textfile collector publishes the metrics
written by that job. Prometheus scrapes and stores them over time, evaluates
alert rules, and supplies the data that Grafana queries for its dashboards.

That is the whole monitoring path: a short systemd service definition, a timer,
and an installer.

## Using the tools already there

NUT already knows how to talk to a wide range of UPS devices. Its `upsc`
command produces useful data such as:

```text
battery.charge: 100
battery.runtime: 1200
ups.load: 34
ups.status: OL
output.voltage: 230.0
```

Prometheus expects metrics in a simple text format:

```text
battery_charge 100
battery_runtime 1200
ups_load 34
ups_status{value="OL"} 1
output_voltage 230.0
```

The `nut-exporter.service` unit closes the gap without a compiled or separate
exporter. It runs `upsc` and lets `awk` split NUT's `key: value` lines, replace
dots in keys with Prometheus-friendly underscores, and write metrics.

Numeric values become gauges. Text values such as UPS status are kept in an
info-style metric label. `ups.status: OB`, for example, becomes:

```text
ups_status{value="OB"} 1
```

NUT state flags can then be queried directly in PromQL. An alert can detect a
UPS running on battery with:

```promql
ups_status{value=~".*OB.*"} == 1
```

## Scheduling without another daemon

The exporter is not a persistent process. It is a `Type=oneshot` systemd
service triggered every minute by `nut-exporter.timer`.

systemd schedules the work and records service logs, so the project does not
need a custom retry loop or process supervisor. Collection runs only when
needed. The service clears `/tmp/nut.prom` before each collection, which means
a failed run cannot leave stale values exposed.

Node Exporter's built-in textfile collector reads the resulting file. With Node
Exporter configured with:

```text
--collector.textfile.directory=/tmp
```

the flow looks like this:

```text
UPS -> NUT/upsc -> systemd service -> /tmp/nut.prom
    -> Node Exporter -> Prometheus -> Grafana and alerting
```

Each handoff uses plain text. That makes the pipeline easy to inspect when
something goes wrong.

## Using the metrics

The repository also includes an importable Grafana dashboard, Prometheus alert
rules, and a Bash installer that deploys or updates the systemd units.

The dashboard shows device information along with battery charge, estimated
runtime, UPS load, output voltage, status, firmware, and outlet state.
Prometheus `instance` and `job` variables let one dashboard cover multiple UPS
devices.

The alert rules define conditions worth acting on:

- `OB` means the UPS is on battery and needs immediate investigation.
- `LB` means the battery is low and protected workloads may need graceful
  shutdown.
- `RB` indicates that the battery should be replaced.
- Low runtime and high load point to capacity risk.
- Missing `ups_status` metrics can indicate a problem with monitoring, NUT, or
  UPS connectivity.

Some rules use values reported by the UPS instead of fixed assumptions. The
critical battery-charge rule compares `battery_charge` with
`battery_charge_low`. The output-voltage rule compares the measurement with
`output_voltage_nominal`.

## Local LLM-assisted implementation

`Qwen3.6-35B-A3B`, running through an MLX inference engine on macOS, helped
speed up the implementation. It was used as a development aid while writing
and iterating on the code and configuration.

The deployed monitoring path remains conventional and local to the Linux host:
NUT, `systemd`, `awk`, Node Exporter, Prometheus, and Grafana. The LLM does not
collect UPS data and is not required to install or operate the exporter.

## Inspecting the system

There is no application runtime to maintain, no container image to build, and
no custom exporter process to debug. You can inspect the operational behavior
with familiar commands:

```bash
upsc my-ups@localhost
sudo systemctl status nut-exporter.timer
sudo journalctl -u nut-exporter.service -f
cat /tmp/nut.prom
```

NUT handles hardware compatibility and systemd schedules collection. Node
Exporter exposes the metrics, while Prometheus and Grafana provide the history,
alerts, and views needed to use them. The service file and the generated
`/tmp/nut.prom` file show how the pieces fit together.

The complete project is available at
[github.com/filippobuletto/nut-ups-monitoring](https://github.com/filippobuletto/nut-ups-monitoring).
Take a look at the repository for the service files, dashboard, alert rules,
and installation instructions.
