# Description

[Fluent Bit](https://fluentbit.io) input plugin that collects CPU metrics from Linux hosts.

This plugin **will only work** on hosts running Linux, because it relies on `/proc/stat` files from [Procfs](https://en.wikipedia.org/wiki/Procfs).

# Requirements

- Docker
- Docker image `fluent-beats/fluent-bit-plugin-dev`

# Build
```bash
./build.sh
```

# Test
```bash
./test.sh
 ```

# Design

This plugin was desined to collect data from any mounted Linux `stat` proc file.

It can be used to collect overall CPU or process CPU usage, even if Fluent Bit is running inside a container, which is not achievable using **native** Fluent Bit `cpu` plugin.

> Potentially LXCFS could bypass that without requiring a custom plugin

## Configurations

This input plugin can be configured using the following parameters:

 Key                    | Description                                   | Default
------------------------|-----------------------------------------------|------------------
 interval_sec           | Interval in seconds to collect data           | 1
 interval_nsec          | Interval in nanoseconds to collect data       | 0
 proc_path              | Path to look for net file                     | /proc
 pid                	| Specify the ID (PID) of a running process in the system. By default the plugin monitors the whole system but if this option is set, it will only monitor the given process ID.             												|

